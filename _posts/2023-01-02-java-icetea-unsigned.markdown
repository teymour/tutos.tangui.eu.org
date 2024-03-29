---
title: Error: Cannot grant permissions to unsigned jars
date:  2023-01-02 15:00
layout: post
---

Chez Online, l'accès KVM nécessite l'execution d'un code java via icedtea-web. Pour certains vieux serveurs, un erreur `Cannot grant permissions to unsigned jars` doit être résolue.

Pour se faire, vous pouvez identifier les `jar` qui posent problème via la `java console` :

    ...
    netx: Initialization Error: Could not initialize application. (Fatal: Application Error: Cannot grant permissions to unsigned jars. Application requested security permissions, but jars are not signed.)
    App already has trusted publisher: false
    Jar found at /home/user/.cache/icedtea-web/cache/3/https/192.168.1.1/443/software/avctVMLinux64.jarhas been verified as UNSIGNED
    Jar found at /home/user/.cache/icedtea-web/cache/2/https/192.168.1.1/443/software/avctKVMIOLinux64.jarhas been verified as UNSIGNED
    Jar found at /home/user/.cache/icedtea-web/cache/1/https/192.168.1.1/443/software/avctKVM.jarhas been verified as UNSIGNED

On peut reproduire l'erreur via l'outil `javasigner` :

    $ jarsigner -verify -certs -verbose /home/user/.cache/icedtea-web/cache/3/https/192.168.1.1/443/software/avctVMLinux64.jar

            196 Fri Jul 25 17:49:26 CEST 2014 META-INF/MANIFEST.MF
            259 Mon Jul 24 09:53:16 CEST 2017 META-INF/DELL.SF
            6669 Mon Jul 24 09:53:16 CEST 2017 META-INF/DELL.RSA
             259 Fri Jul 25 17:49:26 CEST 2014 META-INF/AVOCENT.SF
            1058 Fri Jul 25 17:49:26 CEST 2014 META-INF/AVOCENT.DSA
               0 Tue Nov 05 14:49:36 CET 2013 META-INF/
     m  ? 371353 Tue May 17 15:52:22 CEST 2011 libavmlinux.so
    
      s = signature was verified 
      m = entry is listed in manifest
      k = at least one certificate was found in keystore
      ? = unsigned entry
    
    - Signed by "CN=Avocent, OU=iBMC, O=Avocent Corporation, L=Sunrise, ST=Florida, C=US"
        Digest algorithm: SHA1 (disabled)
        Signature algorithm: SHA1withDSA (disabled), 1024-bit key (weak)
    - Signed by "CN=Dell Inc., O=Dell Inc., L=Round Rock, ST=Texas, C=US"
        Digest algorithm: SHA1 (weak)
        Signature algorithm: MD5withRSA (disabled), 2048-bit key
      Timestamped by "CN=Certum EV TSA SHA2, OU=Certum Certification Authority, O=Unizeto Technologies S.A., C=PL" on lun. juil. 24 09:53:17 UTC 2017
        Timestamp digest algorithm: SHA-1 (weak)
        Timestamp signature algorithm: SHA256withRSA, 2048-bit key
    
    WARNING: The jar will be treated as unsigned, because it is signed with a weak algorithm that is now disabled by the security property:
    
      jdk.jar.disabledAlgorithms=MD2, MD5, RSA keySize < 1024, DSA keySize < 1024, SHA1 denyAfter 2019-01-01

Il faut donc intervenir sur l'option `jdk.jar.disabledAlgorithms` de `java` pour réactiver les éléments désactivé par défaut. Ici `SHA1` et `SHA1withDSA`.

Pour le faire, il faut éditer le fichier `java.security` de la version de votre java.

La version de mon java étant 17.0.5 :

    $ java -version
    openjdk version "17.0.5" 2022-10-18
    OpenJDK Runtime Environment (build 17.0.5+8-Debian-2)
    OpenJDK 64-Bit Server VM (build 17.0.5+8-Debian-2, mixed mode, sharing)

il faut modifier la configuration qui se trouve dans `/usr/lib/jvm/java-1.17.0-openjdk-amd64/conf/security/java.security` :

    #jdk.jar.disabledAlgorithms=MD2, MD5, RSA keySize < 1024, DSA keySize < 1024, SHA1 denyAfter 2019-01-01
    jdk.jar.disabledAlgorithms=MD2, MD5, RSA keySize < 1024, DSA keySize < 1024


`javasigner` indique maintenant que le jar est vérifié :

    $ jarsigner -verify -certs -verbose /home/user/.cache/icedtea-web/cache/3/https/192.168.1.1/443/software/avctVMLinux64.jar
    ...
    
    jar verified.
    
et icetea-web ne pose plus de problème :

    $ javaws viewer.jnlp



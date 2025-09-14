# FreeIPA. Проблема с сертификатом служебного компонента *pki-tomcatd*

> ⚠️
> Проверялось на RHEL 7.4 и Centos 7.4

## Описание проблемы
Служба *pki-tomcatd@pki-tomcat.service* останавливается с ошибкой из-за просроченного сертификата.

Автоматическое и ручное продление сертификата не работает, т.к. используемый сертификат отсутствует в базе PKI.

Вывод getcert list (*certmonger*):

Пример текста ошибки:

```bash 
Request ID '20250709140533': 
       status: MONITORING 
       ca-error: Server at "http://ipa1.test.lan:8080/ca/ee/ca/profileSubmit" replied: Record not found 
       stuck: no 
       key pair storage: type=NSSDB,location='/etc/pki/pki-tomcat/alias',nickname='Server-Cert cert-pki-ca',token='NSS Certificate DB',pin set 
       certificate: type=NSSDB,location='/etc/pki/pki-tomcat/alias',nickname='Server-Cert cert-pki-ca',token='NSS Certificate DB'
```

Т.е. вследствие сбоя сертификат сервиса, находящийся в хранилище NSS DB /var/lib/pki/pki-tomcat/alias и стоящий на мониторинге у сервиса
certmonger, имеет серийный номер, отсутствующий в базе PKI.

## Восстановление работоспособности сервиса pki-tomcatd

### План

1. Получение всех атрибутов просроченного сертификата;
2. Ручной выпуск и подпись нового сертификата с аналогичными атрибутами (через openssl);
3. Помещение нового сертификата и приватного ключа в хранилище NSS DB.

> ⚠️
> Данный способ восстанавливает работу сервиса pki-tomcatd, но не устраняет ошибку "*ca-error: Server at "http://ipa1.test.lan:8080/ca/ee/ca/profileSubmit" replied: Record not found*".
>
> Такой сертификат не обновляется автоматически, поэтому после восстановления работоспособности PKI потребуется перевыпустить сертификат его средствами и поставить на мониторинг в ручном режиме (см. следующую главу).

### Последовательность действий

<details>
<summary>
Нажмите здесь для раскрытия...
</summary>


1. Создаем резервную копию NSS DB:

    ```bash
    cp -ra /var/lib/pki/pki-tomcat/conf/alias/ /var/lib/pki/pki-tomcat/conf/alias_BAK
    ```
       

2. Просматриваем содержимое NSS DB:

    ```bash
    certutil -L -d /var/lib/pki/pki-tomcat/alias
    ```

3. Экспортируем целевой (просроченный) сертификат:

    ```bash
    certutil -L -d /var/lib/pki/pki-tomcat/alias -n "Server-Cert cert-pki-ca" -a > "Server-Cert cert-pki-ca.pem"
    ```

4. Выводим данные о целевом сертификате:

    ```bash
    openssl x509 -in "Server-Cert cert-pki-ca.pem" -text -noout
    ```

5. Экспортируем сертификат центра сертификации (ЦС):

    ```bash
    certutil -L -d /var/lib/pki/pki-tomcat/alias -n "caSigningCert cert-pki-ca" -a > "caSigningCert cert-pki-ca.pem"
    ```

6. Экспортируем приватный ключ центра сертификации (потребуется ввести пароль от хранилища NSS DB, а также задать произвольный ключ шифрования для контейнера pkcs12):

    ```bash
    pk12util -d /var/lib/pki/pki-tomcat/alias -n "caSigningCert cert-pki-ca" -o "caSigningCert cert-pki-ca.p12"
    ```

    > 🔎
    > Пароль от хранилища NSS DB хранится в следующем файле: */var/lib/pki/pki-tomcat/conf/password.conf*.

7. Создаем временный каталог и переходим в него:

    ```bash
    mkdir /root/pki_temp && cd /root/pki_temp
    ```

8. Генерируем приватный ключ для нового сертификата:

    ```bash
    openssl genrsa -out privkey.pem 2048
    ```

9. Извлекаем приватный ключ ЦС из файла pkcs12 (сохранится в PEM-формате в открытом виде):

    ```bash
    openssl pkcs12 -in "caSigningCert cert-pki-ca.p12" -out ca_key.pem -nodes -nocerts
    ```

10. Создаем запрос на выпуск сертификата:

    ```bash
    openssl req -new -key privkey.pem -out Server-Cert_tmp.csr -subj "/O=UNIX.LOCAL/CN=p029i07ipa04.unix.local"
    ```

    > ⚠️
    > Атрибут subject должен быть исправлен под конкретный инстанс FreeIPA, где:
    > 
    > "**/**" — разделитель;
    > 
    > "**O=UNIX.LOCAL**" — realm;
    > 
    > "**CN=p029i07ipa04.unix.local**" — FQDN экземпляра FreeIPA.

11. Выпускаем новый сертификат:

    ```bash
    openssl x509 -req \
    -extfile <(printf "authorityKeyIdentifier=keyid \n authorityInfoAccess=OCSP;URI:http://ipa-ca.unix.local/ca/ocsp \n keyUsage=Digital Signature,Non Repudiation,Key Encipherment,Data Encipherment \n extendedKeyUsage=TLS Web Server Authentication") \
    -days 1825 \
    -in Server-Cert_tmp.csr \
    -CA "caSigningCert cert-pki-ca.pem" \
    -CAkey "ca_key.pem" \
    -set_serial 11005198344 \
    -out Server-Cert_tmp_crt.pem
    ```

    где:

    "**authorityInfoAccess=OCSP;URI:http://ipa-ca.unix.local/ca/ocsp**" — URI до сервиса OCSP (должен совпадать со значением в просроченном сертификате);

    "**-days 1825**" — время валидности сертификата;

    "**-in Server-Cert_tmp.csr**" — сформированный ранее файл запроса на выпуск сертификата;

    "**-CA "caSigningCert cert-pki-ca.pem"**" — сертификат ЦС;

    "**-CAkey "ca_key.pem"**" — приватный ключ ЦС;

    "**-set_serial 11005198344**" — серийный номер сертификата (лучше также взять из просроченного сертификата, т.к. pki-tomcatd при запуске проверяет принадлежность серийного номера к допустимому диапазону);

    "**-out Server-Cert_tmp_crt.pem**" — выпускаемый сертификат.

12. Выводим атрибуты выпущенного сертификата:

    ```bash
    openssl x509 -in Server-Cert_tmp_crt.pem -text -noout
    ```

13. Удаляем просроченный сертификат из NSS DB:

    ```bash
    certutil -D -d /var/lib/pki/pki-tomcat/alias -n "Server-Cert cert-pki-ca"
    ```

14. Конвертируем новый сертификат и его приватный ключ в формат pkcs12 (потребуется задать произвольный ключ шифрования для контейнера pkcs12):

    ```bash
    openssl pkcs12 -inkey privkey.pem -in Server-Cert_tmp_crt.pem -name "Server-Cert cert-pki-ca" -export -out Server-Cert_tmp.p12
    ```

15. Импортируем новый сертификат и его приватный ключ в хранилище NSS DB:

    ```bash
    pk12util -d /var/lib/pki/pki-tomcat/alias -i Server-Cert_tmp.p12 -n "Server-Cert cert-pki-ca"
    ```

16. Проверяем список сертификатов в NSS DB:

    ```bash
    certutil -L -d /var/lib/pki/pki-tomcat/alias
    ```

17. Выводим информацию по сертификату из NSS DB:

    ```bash
    certutil -L -d /var/lib/pki/pki-tomcat/alias -n "Server-Cert cert-pki-ca" -a | openssl x509 -text -noout
    ```

18. Запускаем pki-tomcatd:

    ```bash
    systemctl start pki-tomcatd@pki-tomcat.service
    ```

19. Проверяем статус FreeIPA:

    ```bash
    ipactl status
    ```

</details>

## Перевыпуск и постановка на мониторинг нового сертификата

### План

1. Генерация запроса на выпуск нового сертификата для pki-tomcatd;
2. Создание и импорт нового профиля для выпуска сертификата;
3. Выпуск нового сертификата через утилиту pki (будет присвоен новый серийный номер);
4. Экспорт выпущенного сертификата из PKI;
5. Импорт сертификата и его приватного ключа в NSS DB (*/var/lib/pki/pki-tomcat/alias*);
6. Корректная постановка нового сертификата на мониторинг certmonger.

### Последовательность действий

<details>
<summary>
Нажмите здесь для раскрытия...
</summary>
 
1. Генерируем приватный ключ и запрос на выпуск нового сертификата:

    ```bash
    mkdir /root/kaka_pki && cd /root/kaka_pki
    ```
    
    ```bash
    openssl genrsa -out privkey.pem 2048
    ```
    
    ```bash
    openssl req -new -key privkey.pem -out Server-Cert_tmp.csr -subj "/O=TEST.LAN/CN=ipa1.test.lan"
    ```

    > ⚠️
    > Атрибут subject должен быть исправлен под конкретный инстанс FreeIPA, где:
    >
    > "**/**" — разделитель;
    >
    > "**O=UNIX.LOCAL**" — realm;
    >
    > "**CN=p029i07ipa04.unix.local**" — FQDN экземпляра FreeIPA.

2. Ищем и экспортируем профиль выпуска сертификата "caIPAserviceCert". На его базе будет написан наш профиль.

    ```bash
    pki cert-request-profile-find
    ```
    
    ```bash
    ipa certprofile-show caIPAserviceCert --out=TempCertForTomcatPKI.conf
    ```

    Пример отредактированного профиля **TempCertForTomcatPKI.conf**:

    <details>
    <summary>
    Нажмите здесь для раскрытия...
    </summary>
           
    ```bash
    auth.instance_id=raCertAuth
    classId=caEnrollImpl
    desc=TempCertForTomcatPKI
    enable=true
    enableBy=ipara
    input.i1.class_id=certReqInputImpl
    input.i2.class_id=submitterInfoInputImpl
    input.list=i1,i2
    name=IPA-RA Agent-Authenticated Server Certificate Enrollment
    output.list=o1
    output.o1.class_id=certOutputImpl
    policyset.list=serverCertSet
    policyset.serverCertSet.1.constraint.class_id=subjectNameConstraintImpl
    policyset.serverCertSet.1.constraint.name=Subject Name Constraint
    policyset.serverCertSet.1.constraint.params.accept=truepolicyset.serverCertSet.1.constraint.params.pattern=CN=[^,]+,.+
    policyset.serverCertSet.1.default.class_id=subjectNameDefaultImpl
    policyset.serverCertSet.1.default.name=Subject Name Default
    policyset.serverCertSet.1.default.params.name=CN=$request.req_subject_name.cn$, O=TEST.LAN
    policyset.serverCertSet.11.constraint.class_id=noConstraintImpl
    policyset.serverCertSet.11.constraint.name=No Constraint
    policyset.serverCertSet.11.default.class_id=userExtensionDefaultImpl
    policyset.serverCertSet.11.default.name=User Supplied Extension Default
    policyset.serverCertSet.11.default.params.userExtOID=2.5.29.17
    policyset.serverCertSet.2.constraint.class_id=validityConstraintImpl
    policyset.serverCertSet.2.constraint.name=Validity Constraint
    policyset.serverCertSet.2.constraint.params.notAfterCheck=false
    policyset.serverCertSet.2.constraint.params.notBeforeCheck=false
    policyset.serverCertSet.2.constraint.params.range=740
    policyset.serverCertSet.2.default.class_id=validityDefaultImpl
    policyset.serverCertSet.2.default.name=Validity Default
    policyset.serverCertSet.2.default.params.range=731
    policyset.serverCertSet.2.default.params.startTime=0
    policyset.serverCertSet.3.constraint.class_id=keyConstraintImpl
    policyset.serverCertSet.3.constraint.name=Key Constraint
    policyset.serverCertSet.3.constraint.params.keyParameters=1024,2048,3072,4096
    policyset.serverCertSet.3.constraint.params.keyType=RSA
    policyset.serverCertSet.3.default.class_id=userKeyDefaultImpl
    policyset.serverCertSet.3.default.name=Key Default
    policyset.serverCertSet.4.constraint.class_id=noConstraintImpl
    policyset.serverCertSet.4.constraint.name=No Constraint
    policyset.serverCertSet.4.default.class_id=authorityKeyIdentifierExtDefaultImpl
    policyset.serverCertSet.4.default.name=Authority Key Identifier Default
    policyset.serverCertSet.5.constraint.class_id=noConstraintImpl
    policyset.serverCertSet.5.constraint.name=No Constraint
    policyset.serverCertSet.5.default.class_id=authInfoAccessExtDefaultImpl
    policyset.serverCertSet.5.default.name=AIA Extension Default
    policyset.serverCertSet.5.default.params.authInfoAccessADEnable_0=true
    policyset.serverCertSet.5.default.params.authInfoAccessADLocationType_0=URIName
    policyset.serverCertSet.5.default.params.authInfoAccessADLocation_0=http://ipa-ca.test.lan/ca/ocsp
    policyset.serverCertSet.5.default.params.authInfoAccessADMethod_0=1.3.6.1.5.5.7.48.1
    policyset.serverCertSet.5.default.params.authInfoAccessCritical=false
    policyset.serverCertSet.5.default.params.authInfoAccessNumADs=1
    policyset.serverCertSet.6.constraint.class_id=keyUsageExtConstraintImpl
    policyset.serverCertSet.6.constraint.name=Key Usage Extension Constraint
    policyset.serverCertSet.6.constraint.params.keyUsageCritical=true
    policyset.serverCertSet.6.constraint.params.keyUsageCrlSign=false
    policyset.serverCertSet.6.constraint.params.keyUsageDataEncipherment=true
    policyset.serverCertSet.6.constraint.params.keyUsageDecipherOnly=false
    policyset.serverCertSet.6.constraint.params.keyUsageDigitalSignature=true
    policyset.serverCertSet.6.constraint.params.keyUsageEncipherOnly=false
    policyset.serverCertSet.6.constraint.params.keyUsageKeyAgreement=false
    policyset.serverCertSet.6.constraint.params.keyUsageKeyCertSign=false
    policyset.serverCertSet.6.constraint.params.keyUsageKeyEncipherment=true
    policyset.serverCertSet.6.constraint.params.keyUsageNonRepudiation=true
    policyset.serverCertSet.6.default.class_id=keyUsageExtDefaultImpl
    policyset.serverCertSet.6.default.name=Key Usage Default
    policyset.serverCertSet.6.default.params.keyUsageCritical=true
    policyset.serverCertSet.6.default.params.keyUsageCrlSign=false
    policyset.serverCertSet.6.default.params.keyUsageDataEncipherment=true
    policyset.serverCertSet.6.default.params.keyUsageDecipherOnly=false
    policyset.serverCertSet.6.default.params.keyUsageDigitalSignature=true
    policyset.serverCertSet.6.default.params.keyUsageEncipherOnly=false
    policyset.serverCertSet.6.default.params.keyUsageKeyAgreement=false
    policyset.serverCertSet.6.default.params.keyUsageKeyCertSign=false
    policyset.serverCertSet.6.default.params.keyUsageKeyEncipherment=true
    policyset.serverCertSet.6.default.params.keyUsageNonRepudiation=true
    policyset.serverCertSet.7.constraint.class_id=noConstraintImpl
    policyset.serverCertSet.7.constraint.name=No Constraint
    policyset.serverCertSet.7.default.class_id=extendedKeyUsageExtDefaultImpl
    policyset.serverCertSet.7.default.name=Extended Key Usage Extension Default
    policyset.serverCertSet.7.default.params.exKeyUsageCritical=false
    policyset.serverCertSet.7.default.params.exKeyUsageOIDs=1.3.6.1.5.5.7.3.1
    policyset.serverCertSet.8.constraint.class_id=signingAlgConstraintImpl
    policyset.serverCertSet.8.constraint.name=No Constraint
    policyset.serverCertSet.8.constraint.params.signingAlgsAllowed=SHA1withRSA,SHA256withRSA,SHA512withRSA,MD5withRSA,MD2withRSA,SHA1withDSA,SHA1withEC,SHA256withEC,SHA384withEC,SHA512withEC
    policyset.serverCertSet.8.default.class_id=signingAlgDefaultImpl
    policyset.serverCertSet.8.default.name=Signing Alg
    policyset.serverCertSet.8.default.params.signingAlg=-
    policyset.serverCertSet.list=1,2,3,4,5,6,7,8,11
    profileId=TempCertForTomcatPKI
    visible=false
    ```

    </details>

    > ⚠️
    > Приложенный файл требует редактирования под конкретный инстанс FreeIPA следующих строк:
    >
    > "policyset.serverCertSet.1.default.params.name=CN=$request.req_subject_name.cn$, O=TEST.LAN"
    >
    > "policyset.serverCertSet.5.default.params.authInfoAccessADLocation_0=http://ipa-ca.test.lan/ca/ocsp"

3. Импортируем созданный профиль:

    ```bash
    ipa certprofile-import TempCertForTomcatPKI \
    --file TempCertForTomcatPKI.conf \
    --desc "TempCertForTomcatPKI" \
    --store=true
    ```

4. Получаем билет администратора:

    ```bash
    kinit admin
    ```

5. Выпускаем сертификат на PKI:

    ```bash
    ipa cert-request --principal=dogtag/ipa1.test.lan@TEST.LAN --add --profile=TempCertForTomcatPKI --certificate-out=Server-Cert_tmp.crt Server-Cert_tmp.csr
    ```
    
    где:

    "**--principal=dogtag/ipa1.test.lan@TEST.LAN**" — SPN (можно вывести с помощью команды ipa service-find);
   
    "**--profile=TempCertForTomcatPKI**" — профиль выпуска сертификата (можно вывести с помощью команды pki cert-request-profile-find);
   
    "**--certificate-out=Server-Cert_tmp.crt**" — файл, в котором будет сохранен выпущенный сертификат;
   
    "**Server-Cert_tmp.csr**" — сгенерированный ранее запрос на выпуск сертификата.

    Пример успешного вывода:

    ```bash
    Issuing CA: ipa
    Certificate: ...
    Subject: CN=ipa1.test.lan,O=TEST.LAN
    Subject DNS name: ipa1.test.lan
    Issuer: CN=Certificate Authority,O=TEST.LAN
    Not Before: Tue Jul 29 22:40:04 2025 UTC
    Not After: Fri Jul 30 22:40:04 2027 UTC
    Serial number: 12
    Serial number (hex): 0xC
    ```

7. Сверяем атрибуты выпущенного сертификата с атрибутами старого:

    ```bash
    openssl x509 -text -noout Server-Cert_tmp.crt
    ```
    
    ```bash
    certutil -L -d /var/lib/pki/pki-tomcat/alias -n "Server-Cert cert-pki-ca" -a | openssl x509 -text -noout
    ```
    
8. Бекапим базу NSS DB:

     ```bash
     cp -ra /var/lib/pki/pki-tomcat/conf/alias/ /var/lib/pki/pki-tomcat/conf/alias_BAK
     ```

9. Конвертируем новый сертификат и его приватный ключ в формат pkcs12 (потребуется задать произвольный ключ шифрования для контейнера pkcs12):

     ```bash
     openssl pkcs12 -inkey privkey.pem -in Server-Cert_tmp.crt -name "Server-Cert cert-pki-ca" -export -out Server-Cert_tmp.p12
     ```
     
10. Удаляем целевой сертификат из NSS DB:

     ```bash
     certutil -D -d /var/lib/pki/pki-tomcat/alias -n "Server-Cert cert-pki-ca"
     ```

11. Импортируем новый сертификат и его приватный ключ в базу NSS DB:

    ```bash
    pk12util -d /var/lib/pki/pki-tomcat/alias -i /root/kaka_pki/Server-Cert_tmp.p12 -n "Server-Cert cert-pki-ca"
    ```

12. Проверяем работу pki-tomcatd:

    ```bash
    systemctl restart pki-tomcatd@pki-tomcat.service && ipactl status
    ```
    
13. Ставим сертификат на мониторинг:

    ```bash
    getcert start-tracking \
    -d '/etc/pki/pki-tomcat/alias' \
    -n 'Server-Cert cert-pki-ca' \
    -t 'NSS Certificate DB' -P '$ПАРОЛЬ_ОТ_NSS_DB' \
    -a '/etc/pki/pki-tomcat/alias' \
    -c 'dogtag-ipa-ca-renew-agent' \
    -B '/usr/libexec/ipa/certmonger/stop_pkicad' \
    -C '/usr/libexec/ipa/certmonger/renew_ca_cert "Server-Cert cert-pki-ca"'
    ```
    
    где:
    
    "**-d**" — dir (NSS DB), где хранится сертификат;
    
    "**-n**" — nickname (NSS DB) сертификата;
    
    "**-t**" — token (NSS DB);
    
    "**-a**" — dir (NSS DB), куда будет сохранен сертификат при перевыпуске;
    
    "**-с**" — ЦС для выпуска (вместо ЦС по умолчанию);
    
    "**-B**" — pre-save command;
    
    "**-C**" — post-save command.

    Пример вывода успешной постановки на мониторинг:

    ```bash
    Request ID '20250709140533': 
           status: MONITORING 
           stuck: no 
           key pair storage: type=NSSDB,location='/etc/pki/pki-tomcat/alias',nickname='Server-Cert cert-pki-ca',token='NSS Certificate DB',pin set 
           certificate: type=NSSDB,location='/etc/pki/pki-tomcat/alias',nickname='Server-Cert cert-pki-ca',token='NSS Certificate DB' 
           CA: dogtag-ipa-ca-renew-agent 
           issuer: CN=Certificate Authority,O=TEST.LAN 
           subject: CN=ipa1.test.lan,O=TEST.LAN 
           expires: 2027-06-29 14:04:59 UTC 
           key usage: digitalSignature,nonRepudiation,keyEncipherment,dataEncipherment 
           eku: id-kp-serverAuth 
           pre-save command: /usr/libexec/ipa/certmonger/stop_pkicad 
           post-save command: /usr/libexec/ipa/certmonger/renew_ca_cert "Server-Cert cert-pki-ca" 
           track: yes 
           auto-renew: yes
    ```

> 🔎
> Полезные команды:
>
> "getcert resubmit -i 20250709140533" — перевыпуск сертификата;
>
> "getcert stop-tracking -i 20250709140533" — снять сертификат с мониторинга.

</details>

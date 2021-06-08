### hw3.9

1. Установите [Hashicorp Vault](https://learn.hashicorp.com/vault) в виртуальной машине Vagrant/VirtualBox. Это не является обязательным для выполнения задания, но для лучшего понимания что происходит при выполнении команд (посмотреть результат в UI), можно по аналогии с netdata из прошлых лекций пробросить порт Vault на localhost:

    ```bash
    config.vm.network "forwarded_port", guest: 8200, host: 8200
    ```

    Однако, обратите внимание, что только-лишь проброса порта не будет достаточно – по-умолчанию Vault слушает на 127.0.0.1; добавьте к опциям запуска `-dev-listen-address="0.0.0.0:8200"`.
1. Запустить Vault-сервер в dev-режиме (дополнив ключ `-dev` упомянутым выше `-dev-listen-address`, если хотите увидеть UI).
1. Используя [PKI Secrets Engine](https://www.vaultproject.io/docs/secrets/pki), создайте Root CA и Intermediate CA.
Обратите внимание на [дополнительные материалы](https://learn.hashicorp.com/tutorials/vault/pki-engine) по созданию CA в Vault, если с изначальной инструкцией возникнут сложности.
1. Согласно этой же инструкции, подпишите Intermediate CA csr на сертификат для тестового домена (например, `netology.example.com` если действовали согласно инструкции).
1. Поднимите на localhost nginx, сконфигурируйте default vhost для использования подписанного Vault Intermediate CA сертификата и выбранного вами домена. Сертификат из Vault подложить в nginx руками.
1. Модифицировав `/etc/hosts` и [системный trust-store](http://manpages.ubuntu.com/manpages/focal/en/man8/update-ca-certificates.8.html), добейтесь безошибочной с точки зрения HTTPS работы curl на ваш тестовый домен (отдающийся с localhost). Рекомендуется добавлять в доверенные сертификаты Intermediate CA. Root CA добавить было бы правильнее, но тогда при конфигурации nginx потребуется включить в цепочку Intermediate, что выходит за рамки лекции. Так же, пожалуйста, не добавляйте в доверенные сам сертификат хоста.
1. [Ознакомьтесь](https://letsencrypt.org/ru/docs/client-options/) с протоколом ACME и CA Let's encrypt. Если у вас есть во владении доменное имя с платным TLS-сертификатом, который возможно заменить на LE, или же без HTTPS вообще, попробуйте воспользоваться одним из предложенных клиентов, чтобы сделать веб-сайт безопасным (или перестать платить за коммерческий сертификат).

**Дополнительное задание вне зачета.** Вместо ручного подкладывания сертификата в nginx, воспользуйтесь [consul-template](https://medium.com/hashicorp-engineering/pki-as-a-service-with-hashicorp-vault-a8d075ece9a) для автоматического подтягивания сертификата из Vault.

---
Vagrantfile для запуска виртуалки прилагается.  
Запускаем vault  
```
vault server -dev -dev-listen-address="0.0.0.0:8200"
```
В другой консоли:  
экспортируем ваулт адрес  
```
export VAULT_ADDR=http://127.0.0.1:8200
```
Экспортируем токен Root:  
```
export VAULT_TOKEN=вставляем тут свой токен
```

Включаем pki secret engine:  
```
vault secrets enable pki
```

Устанавливаем срок хранения секретов -1 год.  
```
vault secrets tune -max-lease-ttl=87600h pki
```

Генерим корневой сертификат и сохраняем в CA_cert.crt:  
```
vault write -field=certificate pki/root/generate/internal common_name="example.com" ttl=87600h > CA_cert.crt
```
Настраиваем crl и пути к выданным сертификатам:  
```
vault write pki/config/urls issuing_certificates="$VAULT_ADDR/v1/pki/ca" crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
```
Генерируем Intermediate CA  

Добавляем новый CA с другим путём:
```
vault secrets enable -path=pki_int pki
```
Устанавливаем срок хранения секретов -5 лет.  
```
vault secrets tune -max-lease-ttl=43800h pki_int
```
Делаем intermediate csr  
```
vault write -format=json pki_int/intermediate/generate/internal common_name="example.com Intermediate Authority" | jq -r '.data.csr' > pki_intermediate.csr
```
Подписываем запрос корневым сертификатом:  
```
vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr format=pem_bundle ttl="43800h" | jq -r '.data.certificate' > intermediate.cert.pem
```
Публикуем сертификат:  
```
vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
```
Создаем роль, с которой будем выдавать сертификаты для серверов:  
```
vault write pki_int/roles/example-dot-com allowed_domains="example.com" allow_subdomains=true max_ttl="720h"
```
Создаем сертификат на 5 лет для netology.example.com:  
```
vault write -format=json pki_int/issue/example-dot-com common_name="netology.example.com" alt_names="netology.example.com" ttl="43800h" > netology.example.com.crt
```
```
cat netology.example.com.crt | jq -r .data.certificate > netology.example.com.crt.pem
cat netology.example.com.crt | jq -r .data.issuing_ca >> netology.example.com.crt.pem
cat netology.example.com.crt | jq -r .data.private_key > netology.example.com.crt.key
```
Настраиваем nginx:  
```
mkdir /etc/nginx/ssl
cp netology.example.com.crt.key /etc/nginx/ssl/
cp netology.example.com.crt.pem /etc/nginx/ssl/
```

В /etc/nginx/sites-available/default пишем\добавляем\редактируем:  
```
        listen 443 ssl default_server;
        listen [::]:443 ssl default_server;
        ssl_certificate /etc/nginx/ssl/netology.example.com.crt.pem;
        ssl_certificate_key /etc/nginx/ssl/netology.example.com.crt.key;
        server_name netology.example.com;
```        
В /etc/hosts добавляем:  
```
127.0.0.1    netology.example.com
```

Добавим сертификат intermediate CA в local trust store:  
```
sudo cp intermediate.cert.pem /usr/local/share/ca-certificates/pki_intermediate.crt
sudo update-ca-certificates      
```
Проверяем:
```
curl https://netology.example.com
```
---

Полный вывод из консоли(только токен стер):  
<pre><font color="#4E9A06"><b>vagrant@ubuntu-focal</b></font>:<font color="#3465A4"><b>~</b></font>$ sudo su
root@ubuntu-focal:/home/vagrant# export VAULT_ADDR=http://127.0.0.1:8200
root@ubuntu-focal:/home/vagrant# export VAULT_TOKEN=
root@ubuntu-focal:/home/vagrant# vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/
root@ubuntu-focal:/home/vagrant# vault secrets tune -max-lease-ttl=87600h pki
Success! Tuned the secrets engine at: pki/
root@ubuntu-focal:/home/vagrant# vault write -field=certificate pki/root/generate/internal common_name=&quot;example.com&quot; ttl=87600h &gt; CA_cert.crt
root@ubuntu-focal:/home/vagrant# vault write pki/config/urls issuing_certificates=&quot;$VAULT_ADDR/v1/pki/ca&quot; crl_distribution_points=&quot;$VAULT_ADDR/v1/pki/crl&quot;
Success! Data written to: pki/config/urls
root@ubuntu-focal:/home/vagrant# vault secrets enable -path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/
root@ubuntu-focal:/home/vagrant# vault secrets tune -max-lease-ttl=43800h pki_int
Success! Tuned the secrets engine at: pki_int/
root@ubuntu-focal:/home/vagrant# vault write -format=json pki_int/intermediate/generate/internal common_name=&quot;example.com Intermediate Authority&quot; | jq -r &apos;.data.csr&apos; &gt; pki_intermediate.csr
root@ubuntu-focal:/home/vagrant# vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr format=pem_bundle ttl=&quot;43800h&quot; | jq -r &apos;.data.certificate&apos; &gt; intermediate.cert.pem
root@ubuntu-focal:/home/vagrant# vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
Success! Data written to: pki_int/intermediate/set-signed
root@ubuntu-focal:/home/vagrant# vault write pki_int/roles/example-dot-com allowed_domains=&quot;example.com&quot; allow_subdomains=true max_ttl=&quot;720h&quot;
Success! Data written to: pki_int/roles/example-dot-com
root@ubuntu-focal:/home/vagrant# vault write -format=json pki_int/issue/example-dot-com common_name=&quot;netology.example.com&quot; alt_names=&quot;netology.example.com&quot; ttl=&quot;43800h&quot; &gt; netology.example.com.crt
root@ubuntu-focal:/home/vagrant# cat netology.example.com.crt | jq -r .data.certificate &gt; netology.example.com.crt.pem
root@ubuntu-focal:/home/vagrant# cat netology.example.com.crt | jq -r .data.issuing_ca &gt;&gt; netology.example.com.crt.pem
root@ubuntu-focal:/home/vagrant# cat netology.example.com.crt | jq -r .data.private_key &gt; netology.example.com.crt.key
root@ubuntu-focal:/home/vagrant# mkdir /etc/nginx/ssl
root@ubuntu-focal:/home/vagrant# cp netology.example.com.crt.key /etc/nginx/ssl/
root@ubuntu-focal:/home/vagrant# cp netology.example.com.crt.pem /etc/nginx/ssl/
root@ubuntu-focal:/home/vagrant# vi /etc/nginx/sites-available/default 
root@ubuntu-focal:/home/vagrant# echo &quot;127.0.0.1    netology.example.com&quot; &gt;&gt;/etc/hosts
root@ubuntu-focal:/home/vagrant# sudo cp intermediate.cert.pem /usr/local/share/ca-certificates/pki_intermediate.crt
root@ubuntu-focal:/home/vagrant# sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
root@ubuntu-focal:/home/vagrant# curl https://netology.example.com
curl: (7) Failed to connect to netology.example.com port 443: Connection refused
root@ubuntu-focal:/home/vagrant# systemctl restart nginx
root@ubuntu-focal:/home/vagrant# curl https://netology.example.com
&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;title&gt;Welcome to nginx!&lt;/title&gt;
&lt;style&gt;
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;
&lt;h1&gt;Welcome to nginx!&lt;/h1&gt;
&lt;p&gt;If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.&lt;/p&gt;

&lt;p&gt;For online documentation and support please refer to
&lt;a href=&quot;http://nginx.org/&quot;&gt;nginx.org&lt;/a&gt;.&lt;br/&gt;
Commercial support is available at
&lt;a href=&quot;http://nginx.com/&quot;&gt;nginx.com&lt;/a&gt;.&lt;/p&gt;

&lt;p&gt;&lt;em&gt;Thank you for using nginx.&lt;/em&gt;&lt;/p&gt;
&lt;/body&gt;
&lt;/html&gt;
root@ubuntu-focal:/home/vagrant# 
</pre>

# BÖLÜM-1

## İÇİNDEKİLER:
### **1. Jenkins Kurulumu**
### **2. Java Kurulumu**
### **3. Jenkins Servisini Yönetme**
### **4. Nginx Kurulumu ve Yapılandırması (Reverse Proxy)**
### **5. Self-Signed Sertifika Oluşturma (HTTPS için)**
### **6. Nginx Yapılandırmasını HTTPS için Güncelleme**
### **7. Jenkins'e Erişme**
### **8. Ek Bilgiler; Plugin, Folder, Artifact, Workspace**
---


**1. Jenkins Kurulumu**

- **Jenkins Hakkında Bilgi (Opsiyonel):**

```bash
apt-cache show jenkins
```

Bu komut, Jenkins paketi hakkında bilgi verir (versiyon, bağımlılıklar, açıklama vb.). Kuruluma başlamadan önce Jenkins hakkında bilgi edinmek için faydalıdır.

- **Jenkins Repository Ekleme:**

```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
```

```bash
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Bu adımlar, Jenkins'in resmi paket deposunu sisteminize ekler. Bu sayede, Jenkins'i apt paket yöneticisi ile kurabileceksiniz.

- **Paket Listesini Güncelleme:**

```bash
sudo apt-get update
```

Bu komut, sistemdeki paket listesini günceller. Yeni eklenen Jenkins deposunun da listeye dahil olmasını sağlar.

- **Jenkins Kurulumu:**

```bash
sudo apt-get install jenkins
```

Bu komut, Jenkins'i kurar. Kurulum sırasında bağımlılıklar da otomatik olarak kurulacaktır.

**2. Java Kurulumu**

Jenkins Java ile çalıştığı için, sisteminizde Java'nın kurulu olması gerekir.

- **Paket Listesini Güncelleme (Tekrar):**

```bash
sudo apt update
```

Paket listesini tekrar güncellemek her zaman iyi bir uygulamadır.

- **Java Kurulumu (OpenJDK 17):**
    
    ```bash
    sudo apt install fontconfig openjdk-17-jre
    ```
    
    Bu komut, OpenJDK 17'nin Java Runtime Environment (JRE) sürümünü kurar.  fontconfig, Java uygulamalarının yazı tiplerini doğru şekilde işlemesi için gereklidir. Farklı bir Java sürümü kullanmak isterseniz, openjdk-17-jre yerine istediğiniz sürümü belirtebilirsiniz (örneğin, openjdk-11-jre).
    
- **Java Versiyonunu Kontrol Etme:**
    
    ```bash
    java -version
    ```
    
     Doğru sürümün kurulu olduğunu doğrulamak için önemlidir.
    

**3. Jenkins Servisini Yönetme**

- **Jenkins'i Başlatma:**
    
    ```bash
    sudo systemctl start jenkins
    ```
    
    Jenkins servisini başlatır.
    
- **Jenkins Servis Durumunu Kontrol Etme:**
    
    ```bash
    sudo systemctl status jenkins
    ```
    
- **Jenkins'in Dinlediği Portu Kontrol Etme:**
    
    ```bash
    sudo netstat -ntlp | grep 8080
    ```
    
    Jenkins genellikle 8080 portunda çalışır. Bu komut, 8080 portunun Jenkins tarafından kullanıldığını doğrular. Eğer Jenkins farklı bir portta çalışıyorsa, 8080 yerine o portu kullanın.
    
- **İlk Yönetici Parolasını Alma:**
    
    ```bash
    sudo cat /var/lib/jenkins/secrets/initialAdminPassword
    ```
    
    Bu komut, Jenkins kurulumundan sonra oluşturulan ilk yönetici parolasını gösterir. Bu parolayı, Jenkins'e ilk kez giriş yaparken kullanacaksınız.
    

**4. Nginx Kurulumu ve Yapılandırması (Reverse Proxy)**

Nginx'i bir reverse proxy olarak kullanmak, Jenkins'e güvenli ve kolay bir şekilde erişmenizi sağlar.

- **Paket Listesini Güncelleme:**
    
    ```bash
    sudo apt update
    ```
    
- **Nginx Kurulumu:**
    
    ```bash
    sudo apt install nginx
    ```
    
- **Nginx Servis Durumunu Kontrol Etme:**
    
    ```bash
    sudo systemctl status nginx
    ```
    
- **Firewall Ayarları (UFW):**
    
    ```bash
    sudo ufw allow 'Nginx HTTP'
    sudo ufw allow 'Nginx HTTPS'  #HTTPS için de izin veriyoruz (ileride kullanacağız)
    sudo ufw enable # ufw'yi etkinleştir
    ```
    
    Eğer UFW  kullanıyorsanız, Nginx'in trafiğe izin vermesi için bu komutları çalıştırmanız gerekir.
    
- **Nginx Yapılandırma Dosyası Oluşturma:**
    
    ```bash
    sudo nano /etc/nginx/sites-available/jenkins
    ```
    
    Bu komut, jenkins adında yeni bir Nginx yapılandırma dosyası açar. İçeriği aşağıdaki gibi olmalıdır (alan adınızı/IP adresinizi ve port numarasını kendi değerlerinizle değiştirin):
    
    ```bash
    server {
        listen 80;
        server_name example.com; # Alan adınız veya sunucu IP'niz
    
        location / {
            proxy_pass http://localhost:8080; # Jenkins'in çalıştığı adres ve port
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-Forwarded-Proto $scheme; #HTTPS için önemli
        }
    }
    ```
    
    **Açıklama:**
    
    - “listen 80;” Nginx'in 80 portunu dinlemesini sağlar (HTTP).
    - “server_name example.com;” Alan adınızı veya sunucu IP adresinizi buraya yazın.
    - “proxy_pass http://localhost:8080;” Tüm istekleri localhost:8080 adresine (Jenkins'in çalıştığı adres ve port) yönlendirir.
    - “proxy_set_header ...” Bu satırlar, istemcinin gerçek IP adresini ve diğer bilgileri Jenkins'e iletir. Reverse proxy'nin doğru çalışması için önemlidir.
    - “X-Forwarded-Proto $scheme” HTTPS kullanıyorsanız, Jenkins'e isteğin HTTPS üzerinden geldiğini bildirir.
- **Nginx Yapılandırma Dosyasını Etkinleştirme:**
    
    ```bash
    sudo ln -s /etc/nginx/sites-available/jenkins /etc/nginx/sites-enabled/
    ```
    
    Bu komut, jenkins yapılandırma dosyasını sites-enabled dizinine sembolik link ile ekleyerek etkinleştirir.
    
- **Varsayılan Nginx Yapılandırmasını Devre Dışı Bırakma (Opsiyonel):**
    
    ```bash
    sudo rm /etc/nginx/sites-enabled/default
    ```
    
    Eğer sadece Jenkins'e erişmek istiyorsanız, varsayılan Nginx yapılandırmasını devre dışı bırakabilirsiniz. Ancak, bu sadece tek bir web sitesi barındırıyorsanız geçerlidir. Birden fazla web sitesi barındırıyorsanız, varsayılan yapılandırmayı değiştirmek daha uygun olabilir.
    
- **Nginx Yapılandırmasını Test Etme:**
    
    ```bash
    sudo nginx -t
    ```
    
    Bu komut, Nginx yapılandırma dosyalarınızı sözdizimsel olarak test eder. Hata varsa, hatanın nerede olduğunu gösterir.
    
- **Nginx ve Jenkins'i Yeniden Başlatma:**
    
    ```bash
    sudo systemctl restart nginx
    sudo systemctl restart jenkins
    ```
    
    Yapılandırma değişikliklerinin etkili olması için Nginx ve Jenkins'i yeniden başlatmanız gerekir.
    
- **Servis Durumlarını Kontrol Etme:**
    
    ```bash
    sudo systemctl status nginx
    sudo systemctl status jenkins
    ```
    

**5. Self-Signed Sertifika Oluşturma (HTTPS için)**

Bu adım, Jenkins'e HTTPS üzerinden erişmek için gereklidir. Self-signed sertifika, ücretsiz bir seçenektir. Let's Encrypt gibi bir sertifika otoriteside kullanılabilir.

- **Self-Signed Sertifika Oluşturma:**
    
    ```bash
    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
    ```
    
    Bu komut, 365 gün geçerli bir self-signed sertifika ve özel anahtar oluşturur. Oluşturma sırasında bazı bilgiler istenecektir (ülke, şehir, organizasyon adı vb.).
    
- **Sertifika ve Anahtar Dosyalarının Konumunu Kontrol Etme:**
    
    ```bash
    ls /etc/ssl/certs
    ls /etc/ssl/private
    ```
    

**6. Nginx Yapılandırmasını HTTPS için Güncelleme**

- **Nginx Yapılandırma Dosyasını Düzenleme:**
    
    ```bash
    sudo nano /etc/nginx/sites-available/jenkins
    ```
    
    Dosyanın içeriğini aşağıdaki gibi güncelleyin:
    
    ```bash
    server {
        listen 80;
        server_name example.com; # Alan adınız veya sunucu IP'niz
        return 301 https://$host$request_uri; # HTTP'den HTTPS'ye yönlendirme
    }
    
    server {
        listen 443 ssl;
        server_name example.com; # Alan adınız veya sunucu IP'niz
    
        ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
        ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    
        location / {
            proxy_pass http://localhost:8080; # Jenkins'in çalıştığı adres ve port
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-Forwarded-Proto https;
        }
    }
    ```
    
    **Açıklama:**
    
    - İki server bloğu var: Biri HTTP (80 portu) için, diğeri HTTPS (443 portu) için.
    - HTTP bloğu, tüm istekleri otomatik olarak HTTPS'ye yönlendirir (return 301 https://$host$request_uri;).
    - HTTPS bloğu, SSL'i etkinleştirir (listen 443 ssl;) ve sertifika ve özel anahtar dosyalarının konumlarını belirtir.
    - X-Forwarded-Proto https;: Jenkins'e isteğin HTTPS üzerinden geldiğini bildirir.
- **Nginx Yapılandırmasını Test Etme:**
    
    ```bash
    sudo nginx -t
    ```
    
- **Nginx'i Yeniden Başlatma:**
    
    ```bash
    sudo systemctl restart nginx
    ```
    
- **Firewall'da HTTPS'e İzin Verme (Eğer UFW Kullanıyorsanız):**
    
    ```bash
    sudo ufw allow 'Nginx Full' #Hem HTTP hem de HTTPS'e izin verir.
    ```
    

**7. Jenkins'e Erişme**

Tarayıcınızda https://example.com (veya sunucu IP adresiniz) adresine giderek Jenkins'e erişebilirsiniz. Tarayıcınız, self-signed sertifika nedeniyle bir uyarı gösterebilir. Bu uyarıyı kabul ederek devam edebilirsiniz.


**8. JENKINS**

- **PLUGIN**
    
    Plugin'ler, Jenkins ortamının işlevselliğini kuruluşların veya kullanıcıların özel ihtiyaçlarına göre geliştirmesini sağlayan araçlardır. Bir Jenkins denetleyicisine kurulabilen ve çeşitli yapı araçlarını, bulut sağlayıcılarını, analiz araçlarını ve daha birçok sistemi entegre edebilen binden fazla farklı plugin bulunmaktadır.
    
    Plugin'ler, bağımlılıklarıyla birlikte Güncelleme Merkezi'nden otomatik olarak indirilebilir. Güncelleme Merkezi, Jenkins topluluğunun çeşitli üyeleri tarafından geliştirilen ve sürdürülen açık kaynaklı plugin'lerin bir listesini sunan, Jenkins projesi tarafından işletilen bir hizmettir.
    
    Başta ve daha sonrasında indirilen bu pluginler kaldırılabilir.
    
    - **Dikkat Edilmesi Gerekenler:**
        - Bir plugin'i kaldırmadan önce, o plugin'i kullanan herhangi bir job veya yapılandırma olup olmadığını kontrol edin. Aksi takdirde, kaldırma işlemi sonrasında bu job'lar veya yapılandırmalar düzgün çalışmayabilir.
        - Bazı plugin'ler, diğer plugin'lere bağımlı olabilir. Bir plugin'i kaldırdığınızda, bağımlı olduğu diğer plugin'lerin de çalışmasını etkileyebilirsiniz. Bu nedenle, kaldırma işleminden önce bağımlılıkları kontrol etmek önemlidir.

- **FOLDER**
    
    Jenkins'te çok sayıda job'unuz olduğunda, bunları yönetmek zorlaşabilir. İşte bu noktada "Folder"  özelliği devreye girer.
    
    - **Klasörlerin Amacı:**
        - Klasörler, job'ları mantıksal gruplar halinde düzenlemenize olanak tanır. Örneğin, farklı projeler, ortamlar (test, geliştirme, üretim) veya ekipler için ayrı klasörler oluşturabilirsiniz.
        - Klasörler, job'ları daha kolay bulmanızı, yönetmenizi ve yetkilendirmenizi sağlar.
    - **Klasör Oluşturma:**
        New Item --> Folder
    - **Klasör İçine Job Ekleme:**
        - Var olan bir job'u bir klasöre taşımak için, job'un yapılandırma sayfasına gidin ve "Folder"  seçeneğini kullanarak job'u istediğiniz klasöre taşıyın.
        - Yeni bir job oluştururken, "Folder"  seçeneğini kullanarak job'u doğrudan bir klasörün içinde oluşturabilirsiniz.

- **ARTIFACT**
    
    Jenkins'te bir build işlemi (job çalıştırma) tamamlandığında, ortaya çeşitli sonuçlar çıkabilir. Bu sonuçların en önemlilerinden biri "artifact" olarak adlandırılır.
    
    - **Artifact Nedir?**
        - Artifact, bir build işlemi sonucunda üretilen, dağıtılmaya veya kullanılmaya hazır olan herhangi bir dosyadır.
        - Örnek olarak, bir web uygulamasının WAR dosyası, bir Java uygulamasının JAR dosyası, bir yazılım paketinin kurulum dosyası, test sonuçları raporları veya dokümantasyon dosyaları birer artifact olabilir.
        - Jenkins, artifact'ları otomatik olarak saklayabilir, arşivleyebilir ve paylaşabilir.
    - **Artifact'ları Yönetme:**
        - Jenkins'te, her job için artifact'ların saklanacağı bir dizin belirleyebilirsiniz.
        - "Post-build Actions"  bölümünde, artifact'ları arşivlemek, yayınlamak veya başka bir yere kopyalamak için çeşitli seçenekler mevcuttur.
        ![image](/image/dokuman2_ss/artifact.png)

- **WORKSPACE:**
    
    /varlib/jenkins/workspace Jenkins'in **varsayılan workspace dizinidir.** Jenkins, her job için bu dizin altında ayrı bir alt dizin oluşturur ve job'un tüm verilerini (kaynak kod, build çıktıları, log dosyaları vb.) bu dizinde saklar.
    
    - **Çalışma Alanının Önemi:**
        - Çalışma alanı, Jenkins'in job'ları çalıştırmak için kullandığı temel dizindir.
        - Job'lar, kaynak kodu bu dizine kopyalar, build işlemlerini bu dizinde gerçekleştirir ve artifact'ları bu dizinde oluşturur.
    - **Çalışma Alanını Yönetme:**
        - Jenkins'te, her job için özel bir çalışma alanı dizini belirleyebilirsiniz.
        - Büyük projelerde, disk alanını verimli kullanmak için çalışma alanlarını düzenli olarak temizlemek önemlidir.
        - /var/lib/jenkins/workspace dizinine doğrudan erişmek yerine, Jenkins arayüzünü veya API'sini kullanarak job'ların verilerine erişmek daha güvenlidir.

---

# BÖLÜM-2

## İÇİNDEKİLER:
### **1. Master-Slave Bağlantısı Kurulumu**
### **2. Manuel Build Alma**
### **3. Cron ile Build Alma**
### **4. Trigger ile Build Alma**
### **5. Jenkins User Yönetimi ve Parametrik Job'lar**

---

##  Master-Slave Bağlantısı Kurulumu

Bu adım, Jenkins'in iş yükünü dağıtmak için ikinci bir sanal makine (Slave) oluşturulmasını ve Master makine ile bağlantısının yapılandırılmasını içerir.

**Adımlar:**

1. **İkinci Sanal Makine Kurulumu:** Yeni bir sanal makine oluşturun (Slave).
2. **Şifresiz SSH Bağlantısı:** Master makineden Slave makineye şifresiz SSH bağlantısı kurun.
    - **SSH Anahtarı Oluşturma (Master):**
        
        ```bash
        ssh-keygen -t rsa -b 4096 -C "jenkins server ssh keys" -f jenkins_id_rsa
        ```
        
        Bu komut, Master makinede jenkins_id_rsa  ve jenkins_id_rsa.pub dosyalarını oluşturur.
        
    - **Private Key'i Jenkins'e Ekleme:** Oluşturulan jenkins_id_rsa dosyasının içeriğini Jenkins'in **Credentials** bölümüne ekleyin. Bu, Jenkins'in SSH bağlantısını yönetmesini sağlar.
    - **Public Key'i Slave'e Ekleme:**
        - Slave makinede root kullanıcısına geçin: sudo su - (veya ilgili kullanıcı).
        - .ssh dizini yoksa oluşturun: mkdir -p ~/.ssh
        - authorized_keys dosyası oluşturun veya varsa düzenleyin: nano ~/.ssh/authorized_keys
        - Master makinede oluşturulan jenkins_id_rsa.pub dosyasının içeriğini bu dosyaya yapıştırın.
        - Dosyanın doğru izinlere sahip olduğundan emin olun: chmod 600 ~/.ssh/authorized_keys ve chmod 700 ~/.ssh
    - **SSH Bağlantısını Test Etme:**
        
        ```bash
        ssh -i ./jenkins_id_rsa root@<slave_ip>
        ```
        
         Eğer bağlantı sırasında fingerprint doğrulaması yapılıyorsa kabul edin.
        
3. **SSH Agent Plugin Kurulumu:** Jenkins'in Slave makineye erişmesi için **SSH Agent Plugin**'ini kurun.
4. **Jenkins Job'u Oluşturma ve Test Etme:** Jenkins üzerinden yeni bir Job oluşturun ve Slave makinede çalıştığını doğrulamak için aşağıdaki komutlardan birini çalıştırın:
    
    ```bash
    ip -a
    ifconfig
    cat /etc/hostname
    ```
    

## 1. Manuel Build Alma

**1.Jenkins Job'unu Oluşturun:** 

![image](/image/dokuman2_ss/job-olusturma.png)

**2.SSH Agent'ı Yapılandırın:**

- Job yapılandırmasında **Environment** bölümünde **Use ssh agent** seçeneğini işaretleyin ve önceden oluşturduğunuz credential'ı seçin.

**3.Build Adımları Ekleyin:**

- **Build Steps** bölümünde **Execute shell** adımı ekleyin.
- Slave makinede çalıştırmak istediğiniz komutu bu alana yazın. Örneğin:
        
    ```bash
        ssh -o 'StrictHostKeyChecking=no' root@<slave_ip> '/root/backup.sh'
    ```
        
- StrictHostKeyChecking=no, ilk bağlantıda fingerprint kontrolünü atlar (güvenlik açığı oluşturabileceğini unutmayın).
![image](/image/dokuman2_ss/manuel-build-backıp-script.png)

**4.Backup Script'i Oluşturma (Slave)**

Bu adım, Slave makinesinde çalışacak bir yedekleme script'inin oluşturulmasını içerir.

**4.1. /opt/mysql Klasörü Oluşturma:**
    
```bash
    sudo mkdir -p /opt/mysql
```
    
**4.2./opt/backup Klasörü Oluşturma:**
    
```bash
    sudo mkdir -p /opt/backups
```
    
**4.3.Backup Script'i Oluşturma:** 

/opt/backup.sh dosyasını oluşturun ve aşağıdaki içeriği ekleyin:
    
```bash
    #!/bin/bash
    
    SOURCE_DIR="/opt/mysql"
    BACKUP_DIR="/opt/backups"
    
    mkdir -p "$BACKUP_DIR"
    
    TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
    BACKUP_FILENAME="backup_$(basename "$SOURCE_DIR")_$TIMESTAMP.tar.gz"
    BACKUP_PATH="$BACKUP_DIR/$BACKUP_FILENAME"
    
    if tar -czf "$BACKUP_PATH" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"; then
        echo "Backup successful: $BACKUP_PATH"
    else
        echo "Backup failed!"
        exit 1
    fi
```
    
**4.4.Script'e Çalışma İzni Verme (Slave):**
    
```bash
    sudo chmod +x /opt/backup.sh
```
    
**4.5.Jenkins Job'unu Build Alma:** 

Jenkins'teki ilgili Job'u çalıştırın.

![image](/image/dokuman2_ss/manuel-build-backıp-script-build.png)

## 2. Cron ile Build Alma

Cron ile build alma, Jenkins'teki Job'ların belirli zaman aralıklarında otomatik olarak çalıştırılmasını sağlar.

**Adımlar:**

1. **Job Yapılandırması:** Jenkins'teki ilgili Job'u açın ve yapılandırma sayfasına gidin.
2. **Triggers Bölümü:** "Build Triggers" bölümünü bulun.
3. **Build Periodically:** "Build periodically" seçeneğini işaretleyin.
4. **Zamanlama Ayarı:** Cron ifadesini kullanarak zamanlama ayarlarını yapın. Örneğin, her gece 03:00'te çalıştırmak için: 0 3 * * *

**Not:** Cron ifadelerini Jenkins Job bazlı yönetmek, sunucudaki cron ayarlarını yönetmekten daha kolaydır, özellikle sunucu sayısı arttığında.

## 3. Trigger ile Build Alma

Bu adım, Gitlab gibi versiyon kontrol sistemlerindeki değişikliklerin otomatik olarak Jenkins build'lerini tetiklemesini sağlar.

**Adımlar:**

1. **Gitlab Entegrasyonu:** Jenkins ve Gitlab arasında entegrasyon sağlamak için gerekli eklentileri kurun ve yapılandırın.
2. **Webhook Oluşturma (Gitlab):** Gitlab'ta, ilgili proje için bir webhook oluşturun. Bu webhook, değişiklik olduğunda Jenkins'e bildirim gönderecektir.
    - **URL:** Jenkins sunucusunun URL'si ve Job'un build URL'si. Örneğin: https://jenkins.example.com/job/your_job/build
    - **Secret Token:** Gitlab ve Jenkins arasındaki güvenliği sağlamak için bir secret token kullanın.
3. **Jenkins'te Token Oluşturma:**
    - Jenkins'te kullanıcı hesabınıza gidin.
    - **Configure** sayfasına gidin.
    - **API Token** bölümünde bir token oluşturun.
4. **Gitlab Webhook Ayarları:** Gitlab webhook ayarlarında:
    - **Secret token** alanına oluşturduğunuz token'ı girin.
    - Hangi olayların webhook'u tetikleyeceğini seçin (push, merge request vb.).

**Örnek Curl Komutu (Gitlab'tan Jenkins'i Tetikleme):**

```bash
curl -X POST -L --user your-user-name:apiToken \
https://jenkins.example.com/job/your_job/build
```

- your-user-name: Jenkins kullanıcı adınız.
- apiToken: Jenkins'te oluşturduğunuz API token.
- your_job: Jenkins'teki Job'un adı.

**Not:** Jenkins lokalde çalışıyorsa ve Gitlab'daki branch'lerden tetiklenmesi isteniyorsa, Gitlab'ın da yerelde olması tercih edilir. Alternatif olarak, Cloudflare Tunnel gibi servisler kullanılabilir.

### 6. Cloudflare Tunnel ve Benzeri Servisler

Bu servisler, dışarıdaki bir yapının içerideki bir sunucuya ulaşması gerektiğinde kullanılır.

**Örnek Servisler:**

- **Cloudflare Tunnel:** Güvenli bir tünel oluşturma hizmeti.
- **ngrok:** Yerel sunucuları internete açmak için kullanılır.
- **Pagekite:** Yerel sunucuları internete açmak için kullanılır.
- **Serveo:** SSH tünelleme kullanarak yerel sunucuları internete açar.

## Jenkins User Yönetimi ve Parametrik Job'lar

Bu notlar, Jenkins'te kullanıcı yönetimi, rol tabanlı yetkilendirme ve parametrik job'lar hakkında bilgi içermektedir.

### 1. User Yönetimi

Jenkins'te kullanıcı yönetimi, kimlerin Jenkins'e erişebileceğini ve hangi yetkilere sahip olacağını belirlemeyi sağlar.

**Yöntemler:**

- **Manuel User Oluşturma:**
    - **Admin Paneli:** Manage Jenkins → Users yolunu izleyerek yeni kullanıcılar oluşturabilirsiniz.
    - **Config Dosyası:** Kullanıcı bilgileri /var/lib/jenkins/config.xml dosyasında saklanır. Bu dosyanın yedeklenmesi önemlidir.
- **LDAP/Active Directory Entegrasyonu:** Jenkins'i LDAP veya Active Directory ile entegre ederek kullanıcıları bu dizin servislerinden alabilirsiniz.

**Uygulama Örneği:**

**Manuel User Oluşturma:** 4 adet kullanıcı manuel olarak oluşturuldu.

![image](/image/dokuman2_ss/user-olusturma.png)

**Jenkins Konfigürasyonunu Yedekleme:**

- /var/lib/jenkins/config.xml dosyasını güvenli bir lokasyona kopyalayın.

 - Örneğin: cp /var/lib/jenkins/config.xml /home/jenkins-backup/

- **Yedek Kontrolü:** Dosyaların hash değerlerini alıp karşılaştırarak yedeklemenin başarılı olduğundan emin olun.

**Yetkilendirme Testi:** Bir Job'a sadece okuma yetkisi verin ve gizli modda giriş yaparak yetkilendirmenin doğru çalıştığını kontrol edin.

### 2. Rol Tabanlı Yetkilendirme (RBAC) ve Matris Tabanlı Yetkilendirme

Jenkins’te yetkilendirme, kullanıcıların veya grupların belirli eylemleri gerçekleştirmesine izin vermek için farklı yöntemlerle yönetilebilir. İki yaygın yöntem Role-Based (RBAC) ve Matrix-Based Security’dir.

1️⃣ Role-Based Authorization Strategy (RBAC)

Bu yöntemde kullanıcılara roller atanır ve bu roller belirli yetkilerle ilişkilendirilir. Böylece aynı role sahip kullanıcılar, sistem içinde aynı izinlere sahip olur.

Özellikleri:

✅ Kullanıcıları gruplar halinde yöneterek merkezi kontrol sağlar.
✅ Global, Job ve Agent seviyelerinde roller oluşturulabilir.
✅ Büyük organizasyonlar için uygundur, yeni kullanıcılar sadece ilgili role atanarak yetkilendirilebilir.

![image](/image/dokuman2_ss/role-based-plugin.png)
![image](/image/dokuman2_ss/security-role-base.png)
![image](/image/dokuman2_ss/manage-roles.png)
![image](/image/dokuman2_ss/assign-roles.png)

2️⃣ Matrix-Based Security

Bu yöntemde kullanıcılar veya gruplar doğrudan yetkilendirilir, roller kullanılmaz. Kullanıcı bazında hassas ve detaylı izin ayarı yapmak gerektiğinde kullanışlıdır.

Özellikleri:

✅ Kullanıcı ve gruplara doğrudan izinler atanabilir.
✅ RBAC’ye göre daha esnektir, her kullanıcı için farklı yetki seviyeleri belirlenebilir.
✅ Küçük ekiplerde veya detaylı yetkilendirme gereken durumlarda daha uygundur.

![image](/image/dokuman2_ss/matrix-based-plugin.png)
![image](/image/dokuman2_ss/security-matrix-based.png)


### 3. Parametrik Job'lar

Parametrik Job'lar, kullanıcıların bir build'i başlatırken değerlerini belirleyebileceği parametreler tanımlamanıza olanak tanır. Bu, aynı Job'u farklı konfigürasyonlarla çalıştırmayı kolaylaştırır.

**Adımlar:**

1. **Job Yapılandırması:** İlgili Job'un yapılandırma sayfasına gidin.
2. **This project is parameterized:** "This project is parameterized" seçeneğini işaretleyin.
3. **Parametre Tanımlama:**
    - **String Parameter:** Metin tabanlı değerler için.
    - **Boolean Parameter:** Doğru/Yanlış değerleri için.
    - **Choice Parameter:** Önceden tanımlanmış seçenekler arasından seçim yapmak için.
    - **File Parameter:** Bir dosya yüklemek için.
4. **Parametreleri Kullanma:** Build adımlarında parametre değerlerine $PARAMETRE_ADI şeklinde erişebilirsiniz. Örneğin, bir shell script'te parametre değerini kullanmak için:
    
```bash
    echo "Parametre değeri: $MY_PARAMETER"
```
    
**Örnek Senaryolar:**

- **Ortam Seçimi:** Hangi ortamda (development, staging, production) build alınacağını belirlemek için.
- **Sürüm Numarası:** Uygulama sürümünü belirlemek için.
- **Branch Seçimi:** Hangi Git branch'inin build alınacağını seçmek için.

![image](/image/dokuman2_ss/parametrik-job1.png)
![image](/image/dokuman2_ss/parametrik-job2.png)
![image](/image/dokuman2_ss/parametrik-job3.png)
![image](/image/dokuman2_ss/parametrik-job4.png)
![image](/image/dokuman2_ss/parametrik-job5.png)
![image](/image/dokuman2_ss/parametrik-job6.png)
![image](/image/dokuman2_ss/parametrik-job7.png)
![image](/image/dokuman2_ss/parametrik-job8.png)

**Kaynaklar:**

- **Role Strategy Plugin:** [**https://plugins.jenkins.io/role-strategy/**](https://www.google.com/url?sa=E&q=https%3A%2F%2Fplugins.jenkins.io%2Frole-strategy%2F)
- **Medium Makalesi (User ve Rol Yönetimi):** [**https://medium.com/@srghimire061/how-to-manage-users-and-roles-in-jenkins-fe6a7a8be344**](https://www.google.com/url?sa=E&q=https%3A%2F%2Fmedium.com%2F%40srghimire061%2Fhow-to-manage-users-and-roles-in-jenkins-fe6a7a8be344)



----

# BÖLÜM-3

## İÇİNDEKİLER:
### **1. Pipeline: Stage View Eklentisi Kurulumu**
### **2. Groovy ile Basit Bir Pipeline Oluşturma**
### **3. Jenkins Credential Yönetimi**
### **4. Generate Declarative Directive Kullanımı**
### **5. Jenkins Agent’ları ile Dağıtık Build Ortamları Oluşturma**
### **6. Java Uygulamaları için Tool Ekleme (Maven Örneği)**

---

**1. Pipeline: Stage View Eklentisi Kurulumu**

Bu plugin ile pipeline çıktısını aşamalar halinde görebilir ve her aşamanın durumunu kolayca takip edebiliriz.

**2. Groovy ile Basit Bir Pipeline Oluşturma**

Jenkins arayüzünden Groovy script'i kullanarak basit bir Pipeline oluşturalım.

1. Jenkins ana sayfasında, "New Item—>Pipeline"
2. Bir isim verin (örneğin, "groovy-pipeline").
3. "OK"a tıklayın.
4. Pipeline yapılandırma sayfasında, "Pipeline" bölümüne gidin, menüden "Pipeline script'i" seçeneğini seçin.Aşağıdaki Groovy kodunu metin alanına yapıştırın:

```groovy

pipeline {
    agent any
    stages {
        stage('Merhaba') {
            steps {
                echo 'Merhaba Dünya!'
            }
        }
        stage('Tarih') {
            steps {
                sh 'date'
            }
        }
    }
}
```

5. “Save” ile kaydedin.
6. “Build Now” ile pipeline ı çalıştırın.

![image](/image/dokuman3_ss/pipelinestatus.png)

**Jenkins'te pipeline oluşturmanın iki temel yolu vardır:**

Jenkins Arayüzünde Doğrudan Kodlama ("**Pipeline script**"): Bu yöntemde, pipeline betiğini doğrudan Jenkins işinin yapılandırma sayfasındaki metin alanına yazarsınız. 

SCM'den Pipeline Betiği Çekme ("**Pipeline script from SCM**"): Bu yöntemde, pipeline betiğini (genellikle Jenkinsfile adında) bir versiyon kontrol sisteminde saklarsınız. Jenkins, pipeline'ı çalıştırdığında betiği otomatik olarak SCM'den çeker.

![image](/image/dokuman3_ss/pipeline-olusturma.png)

3. Jenkins Credential Yönetimi

Jenkins'te hassas bilgileri (şifreler, API anahtarları vb.) güvenli bir şekilde saklamak için credential özelliğini kullanabilirsiniz.

Amaç: Pipeline'larda güvenli bir şekilde hassas bilgileri kullanmak.

Adımlar:
Jenkins ana sayfasında, "Manage Jenkins -> Credentials -> System -> Global credentials (unrestricted)" bölümünden add new credential'a tıklayarak yeni credential oluşturun.

![image](/image/dokuman3_ss/credential1.png)

Burada ;

1- Kind: açılır menüden "Secret text" seçeneğini seçin. Bu, gizli bir metin verisi saklayacağımızı belirtir.

2- Scope: Bu, kimlik bilgilerinin tüm Jenkins projelerinde ve işlerinde kullanılabileceği anlamına gelir. Genellikle en iyi seçenektir.
Açıklama: Kimlik bilgilerinin kullanılabilirliğini sınırlamak istiyorsanız, proje veya klasör bazında bir kapsam seçebilirsiniz. Örneğin, yalnızca belirli bir projede kullanılacak bir API anahtarı için, o projeyi seçebilirsiniz.

3- Secret: Buraya, saklamak istediğiniz gerçek gizli metin verisini girin. Bu, gerçekte API anahtarınız, şifreniz veya başka bir hassas bilginiz olacaktır. Bu alana girilen bilgi Jenkins tarafından şifrelenerek saklanacaktır.

4- ID: Bu, credential'ımıza verdiğimiz benzersiz ID'dir. Pipeline kodumuzda bu ID'yi kullanarak credential'a başvuracağız.
Açıklama: Anlamlı ve kolayca hatırlanabilir bir ID seçin. Boşluk veya özel karakterler kullanmaktan kaçının. 

5- Description: Credential'ımızın ne için kullanıldığını açıklayan bir metindir.
Açıklama: Açıklayıcı bir metin girmeniz, gelecekte bu credential'ı kimin veya neyin kullandığını anlamanıza yardımcı olacaktır. Ne tür bir veri sakladığınızı ve nerede kullanıldığını belirtmek faydalı olabilir.

6-  "Create" Butonuna Tıklayın

New Item'dan yeni bir pipeline oluşturun:

````
pipeline {
    agent any

    stages {
        stage('Write Secret to File') {
            steps {
                withCredentials([string(credentialsId: 'Secret_data', variable: 'SECRET_VALUE')]) {
                    script {
                        // Gizli değeri bir dosyaya yaz
                        def secretFilePath = 'secret.txt'
                        writeFile file: secretFilePath, text: "$SECRET_VALUE"

                        // Dosyanın içeriğini konsola yazdır (isteğe bağlı, debug için)
                        //sh "cat ${secretFilePath}"

                        // Dosyanın oluşturulduğunu doğrulamak için bir mesaj yazdır (isteğe bağlı)
                        echo "Secret value has been written to ${secretFilePath}"
                    }
                }
            }
        }
    }
    post {
        always {
            // Dosyayı artifact olarak arşivle
            archiveArtifacts artifacts: 'secret.txt', allowEmptyArchive: true

            // Dosyayı sil (isteğe bağlı, güvenliği artırmak için)
            //deleteDir()
            //echo "Cleanup complete."
        }
    }
}
````
Save butanuna tıklayarak pipeline kaydedebilirsiniz. Ardından "Buil Now" ile çalıştıralım.

![image](/image/dokuman3_ss/credential2.png)
![image](/image/dokuman3_ss/credential3.png)

Aşağıdaki ekran görüntüsü, pipeline tarafından oluşturulan secret.txt adlı artifact'e tıklandığında açılan içeriği göstermektedir. Bu dosya, Jenkins tarafından güvenli bir şekilde saklanan Secret_data adlı credential'dan alınan gizli metni içermektedir.

![image](/image/dokuman3_ss/credential4.png)



4. Generate Declarative Directive Kullanımı

Jenkins'te "Generate Declarative Directive" veya "Pipeline Syntax" olarak bilinen araç, Declarative Pipeline sözdizimini öğrenmeyi ve kullanmayı kolaylaştırmak için tasarlanmıştır. Bu araç, kullanıcı arayüzü üzerinden belirli adımları yapılandırarak, karşılık gelen Declarative Pipeline kodunu otomatik olarak oluşturmanıza olanak tanır.

![image](/image/dokuman3_ss/dec-dir-gen1.png)
![image](/image/dokuman3_ss/dec-dir-gen2.png)
![image](/image/dokuman3_ss/dec-dir-gen3.png)



**Jenkins Agent'ları ile Dağıtık Build Ortamları Oluşturma**

Jenkins agent'ları, Jenkins master sunucusunun iş yükünü dağıtmak ve farklı ortamlarda build işlemleri gerçekleştirmek için kullanılan bağımsız yürütme ortamlarıdır. 

**1. Agent Kavramı ve Önemi:**

Jenkins, build işlemlerini (derleme, test, deployment vb.) gerçekleştirmek için kaynaklara ihtiyaç duyar. Agent'lar, bu kaynakları Jenkins master sunucusundan bağımsız olarak sağlayarak aşağıdaki avantajları sunar:

- **Yük Dengeleme:** Build işlemlerini birden fazla agent'a dağıtarak master sunucunun yükünü azaltır.
- **Farklı Ortamlar:** Farklı işletim sistemleri, araçlar ve bağımlılıklar gerektiren projeler için özel agent'lar oluşturulabilir.
- **Paralel Build:** Birden fazla agent kullanarak build işlemlerini paralel olarak gerçekleştirerek süreci hızlandırır.

**2. Agent Ekleme ve Yapılandırma:**

Agent'lar, Jenkins arayüzünde "Manage Jenkins" -> "Nodes" (veya "Manage Nodes and Clouds") bölümünden eklenir ve yönetilir.

- **Node Türü:** "Permanent Agent" (Kalıcı Agent) veya "Launch agents on demand" (İhtiyaç Duyulduğunda Agent Başlat) seçeneklerinden birini seçin.
- **Agent Adı ve Açıklaması:** Agent'a anlamlı bir ad verin ve açıklamasını ekleyin.
- **Remote root directory:** Agent üzerinde Jenkins'in çalışma alanı olarak kullanacağı bir dizin belirtin. Bu dizin, Jenkins'in agent üzerinde dosyaları saklayacağı ve build işlemlerini gerçekleştireceği yerdir.
- **Labels:** Agent'a etiketler atayın. Bu etiketler, pipeline'larda belirli agent'ları hedeflemek için kullanılır (aşağıdaki bölümlere bakınız).

**3. SSH Anahtar Tabanlı Kimlik Doğrulama ile Agent Bağlantısı:**

Agent'lar, Jenkins master sunucusu ile SSH protokolü üzerinden güvenli bir şekilde iletişim kurar. SSH anahtar tabanlı kimlik doğrulama, şifre tabanlı kimlik doğrulamaya göre daha güvenli bir yöntemdir.

- **3.1. SSH Anahtar Çifti Oluşturma:**
    
    Agent üzerinde aşağıdaki komutu kullanarak bir SSH anahtar çifti oluşturun:
    
    ```groovy
    
    ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
    ```
    
    Bu komut, ~/.ssh dizininde id_rsa (private key) ve id_rsa.pub (public key) olmak üzere iki dosya oluşturur. 
    
- **3.2. Public Key'i Agent'a Yetkilendirme:**
    
    Agent üzerindeki ~/.ssh/authorized_keys dosyasına, public key'in içeriğini ekleyin. Eğer dosya yoksa oluşturun ve doğru izinleri ayarlayın:
    
    ```bash
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh
    touch ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys
    nano ~/.ssh/authorized_keys # public key buraya yapıştırılır
    
    ```
    
    nano veya tercih ettiğiniz bir metin düzenleyici ile ~/.ssh/authorized_keys dosyasını açın ve public key'in içeriğini ( cat ~/.ssh/id_rsa.pub komutu ile elde edebilirsiniz) dosyanın sonuna yapıştırın.
    
- **3.3. Jenkins'te SSH Credential Oluşturma:**
    
    Jenkins'te, "Credentials" bölümüne gidin ve yeni bir credential ekleyin:
    
    - **Kind:** SSH Username with private key
    - **Scope:** Global (veya ihtiyacınıza göre daha kısıtlı bir kapsam)
    - **Username:** Agent üzerinde SSH erişimine sahip olan kullanıcı adı (örneğin, ubuntu, ec2-user, jenkins).
    - **Private Key:** Oluşturduğunuz SSH private key'in içeriğini yapıştırın veya dosyadan yükleyin (~/.ssh/id_rsa).
    - **ID:** Credential'a benzersiz bir ID verin (örneğin, agent_ssh_key).
    - **Description:** Credential'ın ne için kullanıldığını açıklayan bir metin girin.
- **3.4. Agent Konfigürasyonu:**
    
    "Manage Jenkins" -> "Nodes" bölümünde, oluşturduğunuz agent'ı düzenleyin:
    
    - "Launch method" (Başlatma yöntemi) olarak "Launch agents via SSH" seçeneğini seçin.
    - "Host Key Verification Strategy" (Host Anahtarı Doğrulama Stratejisi) olarak "Manually trusted key Verification Strategy" veya "Non verifying Verification Strategy" seçeneğini seçin.
    - "Credentials" (Kimlik Bilgileri) açılır menüsünden, oluşturduğunuz SSH credential'ı seçin.
    - "Host" name alanına agent'ın IP adresini veya hostname'ini yazın.

**4. Pipeline'larda Agent Kullanımı:**

Pipeline'larda agent direktifi ile belirli agent'ları hedefleyebilirsiniz.

- **Label ile Agent Seçimi:**

```
pipeline {
    agent { label 'my-agent' } // "my-agent" etiketine sahip agent'ta çalışır
    stages {
        stage('Build') {
            steps {
                sh 'echo "Build aşaması çalışıyor..."'
            }
        }
    }
}
```
Agent Bağlantısı;

![image](/image/dokuman3_ss/agent-baglantisi.png)

Pipeline'da kullanımı

![image](/image/dokuman3_ss/agent-pipeline.png)
![image](/image/dokuman3_ss/agent-pipeline2.png)

Eğer pipeline'ın tamamının belirli bir agent üzerinde çalışmasını istemiyorsanız, agent none direktifini kullanabilirsiniz. Bu durumda, her bir stage'in kendi agent direktifini içermesi gerekir.

```
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'my-agent' }
            steps {
                sh 'echo "Build aşaması çalışıyor..."'
            }
        }
        stage('Test') {
            agent { label 'another-agent' }
            steps {
                sh 'echo "Test aşaması çalışıyor..."'
            }
        }
    }
}
```

**Java Uygulamaları için Tool Ekleme (Maven Örneği):**

Java uygulamaları için genellikle Maven gibi build araçlarına ihtiyaç duyulur. Jenkins, bu araçları otomatik olarak yükleyebilir veya önceden yüklenmiş araçları kullanabilir.

![image](/image/dokuman3_ss/mvn-tool.png)

- **"Manage Jenkins" -> "Global Tool Configuration"** bölümüne gidin.
- Maven bölümünde, "Maven installations" ekleyin.
- **"Name"** alanına Maven kurulumuna bir isim verin (örneğin, Maven 3.8.1).
- **"Installation directory"** Eğer agent üzerinde maven zaten varsa kurulum dizinini belirtin.
- **"Install automatically"** Eğer maven otomatik kurulacaksa bu seçeneği seçin ve jenkinsin belirtiği url den kurulumu yapın.
- **"Install from local directory"** Eğer lokalde olan bir maven kullanılacaksa bu seçeneği seçin ve dizini belirtin.
- **Spesifik Bir URL'den Çekme:** Jenkins'in belirtilen URL'den Maven'i indirmesini ve kurmasını sağlayabilirsiniz. Bu seçenek, belirli bir Maven sürümüne ihtiyaç duyduğunuzda kullanışlıdır.

Jenkins, build işlemi sırasında gerekli olan Maven'i otomatik olarak agent'a yükleyecektir.

---

# BÖLÜM-4 Jenkins ve Tomcat ile Java Web Uygulaması Dağıtımı

## İÇİNDEKİLER:
### **1. Tomcat 11 Kurulumu**
### **2. Tomcat Kullanıcılarını Yapılandırma**
### **3. Uzaktan Erişimi Yapılandırma (Jenkins ve Tomcat Farklı Sunucularda ise)**
### **4. Tomcat'i Başlat**
### **5. Jenkins'te Gerekli Eklentiyi Yükleme: Deploy to container Plugin**
### **6. Global Tool Configuration (Maven Yapılandırması)**
### **7. Jenkins Pipeline Oluşturma ve Yapılandırma**
### **8. Credential ve Pipeline Entegrasyonu (tomcat-deploy Credential)**
### **9. Pipeline'ı Çalıştırın ve Sonuçları İzleyin**
### **10. Alternatif Deployment Yöntemi: "deploy-war-to-tomcat-without-plugin" (Pluginsiz Dağıtım)**
### **11. Jenkins Configuration as Code (JCasc)**
### **12. Seed Jobs ve Job DSL**

---

**Jenkins ve Tomcat ile Java Web Uygulaması Dağıtımı**

Bu bölümde, bir Java web uygulamasının Jenkins tarafından otomatik olarak Tomcat 11 sunucusuna nasıl dağıtılacağını bakacağız.

**Tomcat 11 Kurulumu:**

Aşağıdaki adımlar, Tomcat 11'in /opt dizinine kurulmasını göstermektedir:

1. **Dizine Git:**
    
    ```bash
    
    cd /opt
    ```
    
2. **Tomcat 11'i İndir:**
    
    ```bash
    
    wget https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.3/bin/apache-tomcat-11.0.3.tar.gz
    ```
    
3. **Arşivi Çıkar:**
    
    ```bash
    
    tar -xf apache-tomcat-11.0.3.tar.gz
    ```
    

**Tomcat Kullanıcılarını Yapılandırma:**

Tomcat'in web arayüzüne erişim ve uygulamaların dağıtımı için kullanıcıları yapılandırmanız gerekmektedir.

1. **tomcat-users.xml Dosyasını Düzenle:**

```bash

nano /opt/apache-tomcat-11.0.3/conf/tomcat-users.xml
```

1. **Kullanıcıları Ekle:**
    
    Aşağıdaki kod bloğunu tomcat-users.xml dosyasına ekleyin ( <tomcat-users> etiketi içinde):
    
    ```bash
    
    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    <role rolename="manager-jmx"/>
    <role rolename="manager-status"/>
    
    <user username="admin"
          password="admin"
          roles="manager-gui,manager-script,manager-jmx,manager-status"/>
    
    <user username="deployer"
          password="deployer"
          roles="manager-script"/>
    
    <user username="tomcat"
          password="s3cret"
          roles="manager-gui"/>
    ```
    
    - admin: Tomcat web arayüzüne tam erişim sağlar.
    - deployer: Uygulamaların dağıtımı için kullanılır (Jenkins ile entegrasyon için önerilen kullanıcı).
    - tomcat: Yalnızca Tomcat web arayüzüne erişim sağlar.
    
    **Güvenlik Notu:** Üretim ortamında, varsayılan kullanıcı adlarını ve parolalarını **değiştirin** ve güçlü parolalar kullanın!
    
    **Uzaktan Erişimi Yapılandırma (Jenkins ve Tomcat Farklı Sunucularda ise):**
    
    Eğer Jenkins ve Tomcat **farklı sunucularda** çalışıyorsa, Jenkins'in Tomcat'e uzaktan erişebilmesi için Tomcat'te ek yapılandırma yapmanız gerekmektedir. Bu yapılandırma, Tomcat'in web arayüzüne ve deployment işlevlerine uzaktan erişime izin vermeyi içerir.
    
    1. **context.xml Dosyalarını Bul:**
        
        Tomcat sunucunuzda, aşağıdaki komutu kullanarak aşağıda belirtilen dosyaları  bulun:
        
        - /opt/apache-tomcat-11.0.3/webapps/manager/META-INF/context.xml
        - /opt/apache-tomcat-11.0.3/webapps/host-manager/META-INF/context.xml
    
    ```bash
    find / -name context.xml
    ```
    

**context.xml Dosyalarını Düzenle:**

Her iki context.xml dosyasını da bir metin düzenleyici ile açın ve <Context> etiketi içinde aşağıdaki <Valve>etiketini bulun

```bash

<Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|JENKINS_SUNUCUSUNUN_IP_ADRESI"/>
```

**JENKINS_SUNUCUSUNUN_IP_ADRESI**: Bu kısmı, **Jenkins sunucunuzun gerçek IP adresiyle** değiştirin. Bu, yalnızca belirli bir IP adresinden (Jenkins sunucusu) Tomcat'in yönetici arayüzüne erişime izin verecektir. Birden fazla IP adresine izin vermek için, adresleri | karakteriyle ayırabilirsiniz.(test için allow=“.*” yapılandırabilirsiniz fakat bu şekilde Tomcat sunucusunun yönetim arayüzünü (manager ve host-manager uygulamaları) **herhangi bir IP adresinden** gelen isteklere açmış olursunuz.)

**Örnek (Tek IP Adresi):**

```bash

<Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192.168.1.100"/>
```

Bu örnek, hem yerel erişime (127.0.0.1 ve ::1) hem de 192.168.1.100 IP adresinden erişime izin verir.

**Tomcat'i Başlat:**

```bash
cd /opt/apache-tomcat-11.0.3/bin/
./startup.sh
```

- Tomcat, güvenlik nedeniyle varsayılan olarak web arayüzüne (manager uygulaması) ve yönetim işlevlerine yalnızca yerel (localhost) erişime izin verir. Bu, Tomcat'in yetkisiz erişime karşı korunmasını sağlar.
- Eğer Jenkins ve Tomcat farklı sunucularda çalışıyorsa, Jenkins'in Tomcat'e uzaktan dağıtım yapabilmesi için, Tomcat'in Jenkins'in IP adresinden gelen isteklere **açıkça** izin vermesi gerekir.
- <Valve> etiketi ve özellikle allow özelliği, Tomcat'e hangi IP adreslerinden gelen isteklere izin vereceğini belirtmek için kullanılır.
- allow özelliğine **Jenkins'in çalıştığı VM'nin IP adresini** ekleyerek, Tomcat'e yalnızca Jenkins'in yönetici arayüzüne erişebileceğini ve dağıtım yapabileceğini söylersiniz.

**Jenkins'te Gerekli Eklentiyi Yükleme: Deploy to container Plugin (** Plugin ile dağıtım yöntemi için gereklidir.)

**Global Tool Configuration (Maven Yapılandırması, dokuman3 de burayı yapmıştık. Burada proje içinde kullanılacak mvn versiyonu aynı olmalı)**

**Jenkins Pipeline Oluşturma ve Yapılandırma:**

Jenkins'te, Tomcat'e dağıtımı otomatikleştirecek bir pipeline oluşturun:

1. **Yeni Pipeline İşi Oluşturun:**
    - Jenkins ana sayfanızda "New Item" seçeneğine tıklayın.
    - Bir isim verin (örneğin, tomcat-deploy-approve).
    - "Pipeline" türünü seçin ve "OK" butonuna tıklayın.
2. **Pipeline Tanımını Yapılandırın:**
    - İş yapılandırma sayfasında, "Pipeline" bölümüne gidin.
    - "Definition" açılır menüsünden "Pipeline script" seçeneğini seçin.
    - pipeline kodunu yazın:
    
    ```groovy
    
    pipeline {
        agent {
            label 'linux'  // Bu pipeline'ın "linux" etiketine sahip bir agent üzerinde çalışacağını belirtir.
        }
        tools {
            maven 'maven 3.8.6' // Bu pipeline'ın "maven 3.8.6" adında bir Maven kurulumu kullanacağını belirtir.
        }
        stages {
            stage('Checkout Code') {
                steps {
                    checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://gitlab.com/ozguryazilimkiskampi/tomcat-deployment-example.git']]])
                }
            }
            stage('Test') {
                steps {
                    sh 'mvn test'
                }
            }
            stage('Build') {
                steps {
                    sh 'mvn package'
                }
            }
            stage("Deploy on Test") {
                steps {
                    deploy adapters: [
                        tomcat9(credentialsId: 'tomcat-deploy',
                         path: '', url: 'http://192.168.186.145:8080')],
                         contextPath: '', war: '**/*.war'
    
                }
    
            }
        }
    }
    ```
    
    - agent { label 'linux' }: Pipeline'ın linux etiketine sahip bir agent üzerinde çalışacağını belirtir. Agent yapılandırması için önceki bölümlere bakın.
    - tools { maven 'maven 3.8.6' }: Pipeline'ın maven 3.8.6 adında bir Maven kurulumu kullanacağını belirtir. Global Tool Configuration bölümünde bu kurulumun yapılandırıldığından emin olun.
    - checkout: Git repository'sinden kaynak kodunu çeker.
    - mvn test: Maven ile testleri çalıştırır.
    - mvn package: Maven ile projeyi paketler (WAR dosyası oluşturur).
    - deploy: "Deploy to container Plugin" eklentisini kullanarak WAR dosyasını Tomcat sunucusuna dağıtır.
        - credentialsId: 'tomcat-deploy': Tomcat credential'ının ID'sini belirtir (önceki adımlarda oluşturduğunuz).
        - url: 'http://192.168.186.145:8080': Tomcat sunucusunun URL'sini belirtir.
        - war: '**/*.war': Dağıtılacak WAR dosyasının yolunu belirtir.
3. **"Save" Butonuna Tıklayarak Pipeline'ı Kaydedin.**

**Credential ve Pipeline Entegrasyonu (tomcat-deploy Credential):**

![image](/image/dokuman4_ss/tomcat-deploy-cred.png)

- **Credential Kimlik Bilgilerini Kontrol Edin:** tomcat-deploy credential'ının, Tomcat sunucusuna bağlanmak için doğru kullanıcı adı ve parolaya sahip olduğundan emin olun. Jenkins -> Credentials bölümünden bu credential'ı kontrol edebilirsiniz.
    - username: deployer (veya Tomcat'te oluşturduğunuz dağıtım kullanıcısı)
    - password: Kullanıcının parolası
- **Pipeline Kodunda Credential ID'sini Doğru Kullanın:**
    
    Pipeline kodunuzda, deploy adımında doğru credentialsId değerini kullandığınızdan emin olun (tomcat-deploy).
    

**Pipeline'ı Çalıştırın ve Sonuçları İzleyin:**

- Pipeline'ı manuel olarak başlatmak için iş sayfasına gidin ve "Build Now" butonuna tıklayın.
- Pipeline'ın konsol çıktısını izleyerek, her aşamanın başarıyla tamamlandığından emin olun.
- Dağıtım başarılı olursa, web uygulamanızın Tomcat sunucusunda çalıştığını doğrulayın.

**Alternatif Deployment Yöntemi: "deploy-war-to-tomcat-without-plugin" (Pluginsiz Dağıtım):**

Bazı durumlarda, belirli bir plugin kullanmak yerine, daha esnek bir çözüm tercih edebilirsiniz. Bu örnekte, "Deploy to container Plugin" kullanmadan, doğrudan SSH üzerinden Tomcat sunucusuna dağıtım yapacağız.

**Ek Credential Oluşturun (SSH Key ile Kimlik Doğrulama):**

Pluginsiz dağıtım için, Tomcat sunucusuna SSH ile bağlanmak için bir credential oluşturmanız gerekmektedir. Bu credential, bir SSH private key içermelidir.

- **SSH Anahtar Çifti Oluşturun:**
    - Tomcat sunucusunda (veya Jenkins sunucusunda), aşağıdaki komutu kullanarak bir SSH anahtar çifti oluşturun:
        
        ```bash
        ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa_deploy
        ```
        
        Bu komut, ~/.ssh dizininde id_rsa_deploy (private key) ve id_rsa_deploy.pub (public key) dosyalarını oluşturur.
        
- **Public Key'i Tomcat Sunucusuna Yetkilendirin:**
    - Tomcat sunucusunda, ~/.ssh/authorized_keys dosyasına public key'in içeriğini ekleyin:
    
    ```bash
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh
    touch ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys
    cat ~/.ssh/id_rsa_deploy.pub >> ~/.ssh/authorized_keys
    ```
    
- **Jenkins'te SSH Credential Oluşturun:**
    - "Credentials" bölümüne gidin ve yeni bir credential ekleyin:
        - **Kind:** SSH Username with private key
        - **Scope:** Global
        - **Username:** Tomcat sunucusunda SSH erişimine sahip olan kullanıcı adı (örneğin, ubuntu, ec2-user, tomcat).
        - **Private Key:** Oluşturduğunuz SSH private key'in içeriğini yapıştırın (~/.ssh/id_rsa_deploy).
        - **ID:** Credential'a benzersiz bir ID verin (örneğin, tomcat_ssh_key).
        - **Description:** Credential'ın ne için kullanıldığını açıklayan bir metin girin (örneğin, "Tomcat sunucusuna SSH erişimi için").
- **Pipeline'ı Oluşturun:**
- **Önemli Not: Nexus Entegrasyonu (Opsiyonel)**

Bu pipeline, üretilen artifact'ı (WAR dosyası) Nexus Repository Manager'a yükleme adımlarını içermektedir. **Nexus kullanımı tamamen opsiyoneldir.** Eğer Nexus kullanmıyorsanız, pipeline'daki "Publish to Nexus" aşamasını **kaldırabilir** veya devre dışı bırakabilirsiniz.  **Nexus Kullanılıyorsa "Nexus Artifact Uploader" plugin’in kurmak gerekir.**

```bash

pipeline {
    
    agent {
        label 'linux'
    }

    environment {
        TOMCAT_SERVER = '<tomcat_ip>'
        TOMCAT_PATH = '/opt/tomcat/webapps'
        TOMCAT_SERVER_USER = 'root'
        TOMCAT_STARTUP_SCRIPT = '/opt/tomcat/bin/startup.sh'
        TOMCAT_SHUTDOWN_SCRIPT = '/opt/tomcat/bin/shutdown.sh'
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "<nexus_ip>:<port>"
        NEXUS_REPOSITORY = "maven-releases"
        NEXUS_CREDENTIAL_ID = "nexus-credentials"
    }
    
    tools {
        maven 'maven 3.8.6'
    }
    
    stages {
        stage('Checkout Code'){
            steps{
                checkout scm
            }
        }
        
        stage('Test'){
            steps{
                sh 'mvn test'
            }
        }
        
        stage('Build'){
            steps{
                sh 'mvn package'
            }
        }
        
        stage('Deploy Onayı') {
            steps {
                input m
                essage: 'Uygulamayı test ortamına deploy etmek istiyor musunuz?',
                      ok: 'Evet, Deploy Et'
            }
        }
        
        stage("Deploy on Test") {
            steps {
                sshagent(['deploy-ssh-key']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no target/*.war ${TOMCAT_SERVER_USER}@${TOMCAT_SERVER}:${TOMCAT_PATH}
                        ssh -o StrictHostKeyChecking=no ${TOMCAT_SERVER_USER}@${TOMCAT_SERVER} "sudo sh ${TOMCAT_SHUTDOWN_SCRIPT}"
                        ssh -o StrictHostKeyChecking=no ${TOMCAT_SERVER_USER}@${TOMCAT_SERVER} "sudo sh ${TOMCAT_STARTUP_SCRIPT}"
                    '''
                }
            }
        }
        
        stage("Publish to Nexus") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml"
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                    artifactPath = filesByGlob[0].path
                    artifactExists = fileExists artifactPath

                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}"
                        
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        )
                    } else {
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }
    }
}

```

**Pipeline'ın İşleyişi:**

1. **Checkout Code:** Kaynak kodunu Git deposundan çeker.
2. **Test:** Maven ile testleri çalıştırır.
3. **Build:** Maven ile projeyi paketler (WAR dosyası oluşturur).
4. **Deploy Onayı:** Kullanıcıdan dağıtım için onay ister.
5. **Deploy on Test:**
    - **sshagent(['deploy-ssh-key']):** deploy-ssh-key adlı SSH credential'ını kullanarak SSH bağlantısı kurar. Bu credential, Tomcat sunucusuna bağlanmak için gerekli olan private key'i içerir.
    - **scp -o StrictHostKeyChecking=no target/*.war ${TOMCAT_SERVER_USER}@${TOMCAT_SERVER}:${TOMCAT_PATH}:** WAR dosyasını Tomcat sunucusuna kopyalar.
        - **scp:** SSH üzerinden dosya kopyalama komutu.
        - **o StrictHostKeyChecking=no:** İlk bağlantıda host key'i otomatik olarak kabul eder. (Güvenlik açısından dikkatli olunmalı, gerçek ortamda bu seçenek kaldırılabilir veya dikkatli bir şekilde yönetilmelidir).
        - **target/*.war:** Kopyalanacak WAR dosyasının konumu.
        - **${TOMCAT_SERVER_USER}@${TOMCAT_SERVER}:${TOMCAT_PATH}:** Dosyanın kopyalanacağı hedef sunucu ve dizin.
    - **ssh -o StrictHostKeyChecking=no ${TOMCAT_SERVER_USER}@${TOMCAT_SERVER} "sudo sh ${TOMCAT_SHUTDOWN_SCRIPT}":** Tomcat'i durdurur.
    - **ssh -o StrictHostKeyChecking=no ${TOMCAT_SERVER_USER}@${TOMCAT_SERVER} "sudo sh ${TOMCAT_STARTUP_SCRIPT}":** Tomcat'i başlatır.
6. **Publish to Nexus:** (Opsiyonel) Üretilen artifact'ı (WAR dosyası) Nexus'e yükler. Bu aşama, artifact'ların merkezi bir depoda saklanmasını ve paylaşılmasını sağlar.

**Jenkins Configuration as Code (JCasc)**

Jenkins Configuration as Code (JCasc), Jenkins ayarlarını kod olarak yönetmenizi sağlayan bir yaklaşımdır. Bu yaklaşım, Jenkins ayarlarını YAML formatında dosyalarda saklamanıza ve versiyon kontrol sisteminde  tutmanıza olanak tanır.

JCasc ile Jenkins ortamınızı kolayca yeniden oluşturabilir veya farklı ortamlara dağıtabilirsiniz.Jenkins ayarlarını otomatik olarak uygulayabilir ve yönetebilirsiniz.

Bu özelliği kullanabilmek için Jenkins'te "Configuration as Code" plugin'ini kurun.

![image](/image/dokuman4_ss/conf-as-code.png)

**Seed Jobs ve Job DSL**

Seed Jobs ve Job DSL (Domain-Specific Language), Jenkins'te job’ları otomatik olarak oluşturmanızı sağlayan bir yaklaşımdır. Bu yaklaşım, büyük sayıda benzer işi yönetmek ve yapılandırmayı otomatikleştirmek için idealdir.

**Ek Notlar:**

- JCasc ve Job DSL, Jenkins'i daha ölçeklenebilir, yönetilebilir ve sürdürülebilir hale getirmek için kullanabileceğiniz güçlü araçlardır.
- Bu konular, bu dokümanın kapsamı dışındadır. Ancak, daha fazla bilgi edinmek için aşağıdaki kaynaklara başvurabilirsiniz.
- ****https://www.jenkins.io/projects/jcasc/
- [**https://www.jenkins.io/doc/pipeline/steps/job-dsl/**](https://www.google.com/url?sa=E&q=https%3A%2F%2Fwww.jenkins.io%2Fdoc%2Fpipeline%2Fsteps%2Fjob-dsl%2F)
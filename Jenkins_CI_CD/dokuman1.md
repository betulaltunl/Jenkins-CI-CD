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

**JENKINS - JAVA - SELF CRT - NGINX - NGINX REVERSE PROXY KURULUM VE YAPILANDIRMASI**

Bu kılavuz, bir Ubuntu (veya benzeri Debian tabanlı) sunucuya Jenkins, Java, Self-Signed Sertifika ve Nginx Reverse Proxy kurulumunu ve yapılandırmasını kapsar.

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

**Önemli Notlar:**

- **Güvenlik:** Self-signed sertifikalar güvenli değildir. Üretim ortamında Let's Encrypt gibi bir sertifika otoritesinden sertifika almanız önerilir.
- **Alan Adı:** example.com yerine kendi alan adınızı (veya sunucu IP adresinizi) kullanın.
- **Port Numarası:** Jenkins'in varsayılan portu 8080'dir. Eğer Jenkins farklı bir portta çalışıyorsa, Nginx yapılandırmasında doğru portu belirtin.
- **Dosya İzinleri:** Bazı durumlarda, dosya izinleriyle ilgili sorunlar yaşayabilirsiniz. Gerekirse, ilgili dosyalara doğru izinleri verin.
- **Güncelleme:** Jenkins, Java ve Nginx'i düzenli olarak güncel tutulmalıdır.

**JENKINS**

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

        
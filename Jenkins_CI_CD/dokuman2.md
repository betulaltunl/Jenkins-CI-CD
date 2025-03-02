# BÖLÜM-2

## İÇİNDEKİLER:
### **1. Master-Slave Bağlantısı Kurulumu**
### **2. Manuel Build Alma**
### **3. Cron ile Build Alma**
### **4. Trigger ile Build Alma**
### **5. Jenkins User Yönetimi ve Parametrik Job'lar**

---

# Jenkins Build Alma Notları

Bu notlar, Jenkins üzerinde build alma süreçlerini ve konfigürasyonlarını kapsamaktadır.

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
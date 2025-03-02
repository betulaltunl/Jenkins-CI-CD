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
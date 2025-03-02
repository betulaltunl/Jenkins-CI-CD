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
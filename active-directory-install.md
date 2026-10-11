# Active Directory Lab: Kurulum ve Saldırı Rehberi

Bu rehber, sıfırdan bir Active Directory lab ortamı kurmayı ve bu ortamda gerçek dünya sızma testi senaryolarını uygulamayı adım adım anlatır.

---

##  İçindekiler

- [Windows Server Kurulumu ve Active Directory Yapılandırması](#bolum-1)
  - [1.1. Windows Server Sürüm Seçimi](#bolum-1-1)
  - [1.2. Active Directory Öncesi İki Altın Kural](#bolum-1-2)
  - [1.3. Active Directory Rolünü Ekleme](#bolum-1-3)
  - [1.4. Domain Controller Terfisi](#bolum-1-4)
  - [1.5. Install Sonrası Arka Planda Ne Olur?](#bolum-1-5)
  - [1.6. Kurulumu Doğrulama](#bolum-1-6)
  - [1.7. ADUC ile Kullanıcı, Grup ve OU Yönetimi](#bolum-1-7)
  - [1.8. İstemci Makineleri Domain'e Katma](#bolum-1-8)

<a id="bolum-1"></a>
# Bölüm 1: Windows Server Kurulumu ve Active Directory Yapılandırması

<a id="bolum-1-1"></a>
## 1.1. Windows Server Sürüm Seçimi

Active Directory laboratuvarı kuracağımız ve görsel arayüz üzerinden yönetim/analiz yapacağımız için **Desktop Experience** zorunludur.

Listede 4 seçenek bulunur. Bunların farkları ve kullanım amaçları şunlardır:

### "Desktop Experience" Olan vs Olmayan Seçenekler

- **Desktop Experience (Masaüstü Deneyimi):** Windows'un alıştığın pencereli, fareyle kontrol edilen grafik arayüzünü (GUI) kurar. Server Manager, Active Directory Users and Computers (ADUC), DNS/DHCP konsolları gibi yönetim araçlarını grafiksel olarak kullanmanı sağlar. Eğitim, siber güvenlik lab ortamları ve görsel yönetim için bu seçilir.

- **Desktop Experience Yazmayanlar (Server Core):** Grafik arayüzü tamamen iptal eder. Sistem açıldığında karşına sadece siyah bir CMD / PowerShell ekranı gelir.

  **Neden kullanılır?** RAM ve işlemci tüketimi çok düşüktür, diskte az yer kaplar ve grafik bileşenleri olmadığı için saldırı yüzeyi (attack surface) daha dardır. Kurumsal prodüksiyon ortamlarında sadece PowerShell veya uzaktan yönetim araçlarıyla yönetilen sunucularda tercih edilir.

### Standard vs Datacenter Sürümleri

- **Standard Evaluation:** Orta ve küçük ölçekli yapılar için tasarlanmıştır. Fiziksel bir lisans alındığında üzerinde sadece 2 adet sanal makine (VM) çalıştırma hakkı verir.

- **Datacenter Evaluation:** Büyük kurumlar, bulut altyapıları ve yoğun sanallaştırma yapılan ortamlar içindir. Üzerinde sınırsız sayıda sanal makine çalıştırma hakkı, gelişmiş depolama özellikleri (Storage Spaces Direct) ve korumalı sanal makineler (Shielded VMs) gibi kurumsal özellikler barındırır.

- **Bizim için farkı ne?** Tek bir sanal makinede Active Directory labı yapacağımız için ikisi de Active Directory rolünü birebir aynı çalıştırır. Ancak lab ortamında tüm enterprise özellikleri açık tutmak adına **Datacenter Evaluation (Desktop Experience)** seçmek en idealidir.

### Nasıl Devam Edeceksin?

<img width="915" height="619" alt="Ekran Görüntüsü 2026-10-08 20-35-47" src="https://github.com/user-attachments/assets/81ff2fca-6bb7-4a64-bad5-9f5a09440005" />

1. Listeden **Windows Server 2016 Datacenter Evaluation (Desktop Experience)** satırını seç.
2. Sağ alttaki **Next** butonuna tıkla.
3. Lisans şartlarını kabul et (**Accept the license terms**).
4. Kurulum türü sorulduğunda **Custom: Install Windows only (advanced)** seçeneğini seç.
5. Disk seçim ekranında VirtualBox'ta ayırdığımız diski seçip doğrudan **Next** de. Kurulum dosyaları kopyalanmaya başlayacaktır.

Sistem kurulup yeniden başladığında senden bir **Administrator parolası** isteyecek. Windows Server varsayılan olarak katı parola karmaşıklığı (password complexity) kuralları uygular. En az 8 karakterli, büyük harf, küçük harf, rakam ve özel karakter içeren bir parola belirle (Örnek: `P@ssw0rd123!` veya `LabAdmin2026!`).


### Giriş ve Misafir Eklentileri

- Kilit ekranında "Ctrl + Alt + Delete" basmanı isterse fiziksel klavyenden basma (ana Arch sistemine gider). Bunun yerine VirtualBox penceresinin üst menüsünden **Giriş > Klavye > Ctrl-Alt-Del Ekle** (veya klavyenden **Sağ Ctrl + Del**) kombinasyonunu kullan.

- Masaüstü açıldığında üst menüden **Aygıtlar > Misafir Eklentileri CD Kalıbını Yerleştir** (Insert Guest Additions CD image) seçeneğine tıkla. Dosya Gezgini'nden sürücüye girip `VBoxWindowsAdditions.exe` dosyasını kur ve sistemi yeniden başlat (bu işlem tam ekran çözünürlüğünü ve fare entegrasyonunu sağlar).

---

<a id="bolum-1-2"></a>
## 1.2. Active Directory Öncesi İki Altın Kural

Active Directory Domain Services (AD DS) bir sunucuya kurulduktan sonra makinenin adını veya IP adresini değiştirmek etki alanının iç yapısını bozabilir. Bu yüzden rolü eklemeden önce şunları tamamlamalıyız:

### 1. Sunucu Adını Değiştir (Computer Name)

- Açılan **Server Manager** penceresinde sol panelden **Local Server** seçeneğine tıkla.
- **Computer name** (örneğin `WIN-XXXXXXX` gibi rastgele bir isimdir) üzerine tıkla.
- **Change (Değiştir)** butonuna basarak sunucuna lab ortamına uygun net bir isim ver (Örn: `DC01` veya `PRIMARY-DC`).
- Tamam deyip kapat (sistem yeniden başlatma isteyecektir, yeniden başlat).


### 2. Statik IP Yapılandır

- Bir etki alanı denetleyicisi (Domain Controller) aynı zamanda ağın DNS sunucusu olacağı için kesinlikle dinamik IP (DHCP) almamalıdır.

- `Win + R` yapıp `ncpa.cpl` yaz ve Enter'a bas.

- Ağ bağdaştırıcısına sağ tıklayıp **Properties (Özellikler)** de.

- **Internet Protocol Version 4 (TCP/IPv4)** seçeneğine çift tıkla.

- Statik IP ver (örneğin NAT Network kullanıyorsan: IP: `10.0.2.10`, Subnet: `255.255.255.0`, Gateway: `10.0.2.1`).

- **En önemli kısmı:** Preferred DNS server kısmına **kendi IP adresini** (`10.0.2.10`) veya loopback adresi olan `127.0.0.1` yaz.

<img width="820" height="744" alt="Ekran Görüntüsü 2026-10-09 01-08-17" src="https://github.com/user-attachments/assets/9684f4b7-cb07-49de-bf1d-cf02b189acaf" />

### Neden Bu İki Kural?

**1. Neden Makine İsmini Değiştirdik (WIN-XXXX yerine DC01)?**

Yalnızca karışıklığı önlemek için değil, Active Directory mimarisi kurulduktan sonra isim değiştirmenin çok riskli olması nedeniyle yaptık:

- Bir sunucu Domain Controller (DC) yapıldığında, adı Active Directory veritabanına (**NTDS.dit**), Kerberos biletlerine ve DNS SRV (servis) kayıtlarına kalıcı olarak işlenir.
- Kurulum bittikten sonra makine adını değiştirmeye kalkarsan DNS kayıtları bozulabilir, sertifikalar geçersiz kalabilir ve etki alanındaki istemci makineler sunucuyu bulamayabilir (`netdom` gibi karmaşık komutlar gerekir).
- Bu yüzden kurumsal yapılarda sunucu adı rol kurulmadan önce standart bir şablona (`DC01`, `PDC-LAB`) çekilir.

**2. Statik IP Neden Zorunludur?**

- **Altyapı Taşlarının Sabit Kalması:** Bu sunucu artık ağın "merkezi beyni" (kimlik doğrulama otoritesi ve DNS sunucusu) haline gelecek.
- **Kritik Hata Senaryosu:** Eğer IP adresi DHCP üzerinden dinamik kalsaydı; router/modem yeniden başladığında veya kira süresi (lease time) bittiğinde DC'nin IP'si değişebilirdi. IP değiştiği anda ağdaki yüzlerce istemci bilgisayar domain'e giriş yapamaz, kimlik doğrulayamaz ve kilitlenirdi.

**3. Preferred DNS Server Olarak Kendi IP'sini veya Loopback (127.0.0.1) Yazmak Ne İşe Yarar?**

Active Directory, DNS olmadan kesinlikle çalışamaz.

- Normalde bilgisayarlar DNS olarak Google (8.8.8.8) veya ev modemini kullanır. Ancak Google ya da modem, senin kurduğun yerel etki alanını (örneğin `lab.local`) veya `_ldap._tcp.dc._msdcs...` gibi özel AD servis kayıtlarını bilemez.
- Domain Controller, kurulum sihirbazı bittiğinde kendi bünyesinde bir DNS sunucusu çalıştıracaktır.
- DNS kısmına `127.0.0.1` (veya kendi statik IP'si) yazarak sunucuya şu emri vermiş olduk: *"Bir isim çözmek istediğinde dışarıya sorma; kendi veritabanındaki DNS kayıtlarına bak, çünkü o domain'in tek sahibi sensin."*

---

<a id="bolum-1-3"></a>
## 1.3. Active Directory Rolünü Ekleme

- Yeniden **Server Manager > Dashboard** ekranına dön.
- Ortadaki 2. sıradaki **Add roles and features** butonuna tıkla.
- **Server Roles** adımına kadar "Next" ile ilerle.
- Rol listesinden **Active Directory Domain Services** kutucuğunu işaretle, açılan pencerede **Add Features** de ve kurulumu tamamla.

<img width="943" height="757" alt="Ekran Görüntüsü 2026-10-09 01-09-29" src="https://github.com/user-attachments/assets/33df34de-2566-4300-9046-1747409ebdbd" />

Bu işlemi yaptıktan sonra artık Active Directory Domain Controller için gerekli özellikleri yüklemiş oluyoruz.

---

<a id="bolum-1-4"></a>
## 1.4. Domain Controller Terfisi

Server Manager'e geri gelip bayrağın yanındaki ünlem işaretine basıp **"Promote this server to a domain controller"** (Bu sunucuyu etki alanı denetleyicisine yükselt) diyoruz.

### Dağıtım Yapılandırması (Deployment Configuration)

Karşımıza 3 seçenek çıkacak:

1. **Add a domain controller to an existing domain** (Mevcut etki alanına ek DC katma)
2. **Add a new domain to an existing forest** (Mevcut ormana yeni alt etki alanı ekleme)
3. **Add a new forest** (Yeni bir orman ekle)

Sıfırdan bağımsız bir yapı kurduğumuz için alttaki **Add a new forest** seçeneğini işaretle.

**Root domain name:** Etki alanın için bir isim belirle (Örn: `selo.local`, `lab.local` veya `siber.local`). `.local` uzantısı yerel lab ortamları için standarttır. İsim yazıp **Next** de.


### Domain Controller Seçenekleri (Domain Controller Options)

- **Forest and Domain functional level:** Varsayılan olarak Windows Server 2016 kalabilir.
- **DNS Server** ve **Global Catalog (GC)** kutucuklarının işaretli olduğundan emin ol (otomatik gelir).
- **DSRM Şifresi (Directory Services Restore Mode Password):** Active Directory veritabanı çökerse kurtarma moduna girmek için kullanılan yedek şifredir. Buraya yine karmaşık bir parola gir (Örn: az önceki Administrator parolanla aynı yapabilirsin) ve **Next** de.

### DNS Seçenekleri & NetBIOS Adı

- **DNS delegasyonu uyarısını** (*A delegation for this DNS server cannot be created...*) görürsen endişelenme; bu uyarı bir üst otorite internet DNS'i olmadığı için çıkar. Doğrudan **Next** diyerek geç.
- **NetBIOS name:** Belirlediğin domain adının büyük harfli kısa hali (örn. `SELO` veya `LAB`) otomatik gelecektir. Değiştirmeden **Next** de.

### Yollar (Paths) ve Önkoşul Kontrolü

- Veritabanı klasörü (**NTDS**), loglar ve **SYSVOL** paylaşımlarının yolları varsayılan olarak `C:\Windows\...` şeklinde kalabilir. **Next** ile ilerle.
- **Review Options** ekranını da **Next** ile geç.
- Sistem önkoşulları kontrol edecek. En üstte yeşil bir onay işareti ve **"All prerequisite checks passed successfully"** yazısını gördüğünde alttaki **Install** butonuna bas.


Kurulum bittiğinde sunucu otomatik olarak yeniden başlayacak. Makine açıldığında kilit ekranında KULLANICIADI yerine artık **DOMAIN\Administrator** (örneğin `SELO\Administrator`) şeklinde kurumsal oturum açma formatını göreceksin.

---

<a id="bolum-1-5"></a>
## 1.5. Install Sonrası Arka Planda Ne Olur?

Install butonuna bastıktan sonra arka planda şu işlemler gerçekleşir:

- Dizin bölümleri ve şema yapılandırılır,
- Şifrelenmiş kimlik veritabanı dosyası olan **ntds.dit** oluşturulur,
- Domain politikalarının ve betiklerin istemcilere dağıtılmasını sağlayan **SYSVOL** paylaşım klasörü açılır,
- Yerel etki alanı için DNS sunucu kayıtları (**`_ldap`, `_kerberos`** vb. SRV kayıtları) sisteme işlenir.**)

Kurulum tamamlandığında Windows Server sana herhangi bir soru sormadan sistemi otomatik olarak yeniden başlatacaktır. İlk açılış, etki alanı servisleri ilk kez devreye girdiği için normalden biraz daha uzun sürebilir (*"Applying computer settings..."* ekranında biraz beklemesi tamamen olağandır).**)

### Gerçek Dünya Analojisi: 25 Bilgisayarlık Bir Ofis

Gözünde 25 bilgisayarlık bir siber güvenlik şirketinin ofisini canlandır. Ofisteki tüm bilgisayarlar birbirine bağlı ama düne kadar hepsi başıboş gezen birer "Home PC" gibiydi. Kimin hangi bilgisayarda ne yaptığı belirsizdi, herkes kendi makinesinin yerel yöneticisiydi. Sen şimdi bu makinelerden birine Windows Server kurup Domain Controller (DC) yetkisi verdin ve Install butonuna bastın.

Arka planda dönen o teknik kavramların bu ofisteki somut karşılığı şudur:

**1. ntds.dit (Şifrelenmiş Kimlik Veritabanı) — Şirketin Çelik Kasası**

Daha önce ofisteki Ahmet, Mehmet veya Selo sadece kendi masalarındaki bilgisayara giriş yapabiliyordu; çünkü şifreleri o bilgisayarın yerel **SAM (Security Account Manager)** dosyasında tutuluyordu.

- **Ne oldu?** Sen DC kurduğunda, sunucunun diskinde `C:\Windows\NTDS\ntds.dit` adında devasa bir veritabanı kütüğü oluştu.
- **Senaryo:** Artık 25 personelin adı, soyadı, parolalarının NTLM hash'leri, telefonları ve yetki grupları bu dosyanın içine yazılır. Ahmet kendi bilgisayarı bozulduğunda yan masadaki Ayşe'nin bilgisayarına geçip kullanıcı adını ve şifresini yazar. Ayşe'nin bilgisayarı *"Ben seni tanımıyorum"* demez; hemen arka planda senin bu sunucuna ulaşıp `ntds.dit` dosyasına bakar: *"Bu Ahmet bizim şirkette mi, şifresi doğru mu?"* Onayı aldığında Ahmet masasını taşımadan oturumunu açar.
- **Siber Güvenlik Açısı:** Bir pentest veya Red Team operasyonunda `ntds.dit` dosyasını ele geçirmek **"oyunu bitirmek" (Full Domain Compromise)** demektir. Çünkü şirketteki patron dahil herkesin parola hash'i bu dosyanın içindedir.

**2. Dizin Bölümleri ve Şema (Partitions & Schema) — Kurumun Organizasyon Şeması**

25 personeli rastgele bir havuza atamazsın; kimisi muhasebeci, kimisi yazılımcı, kimisi sızma testi uzmanı.

- **Ne oldu?** AD kurulumu dizin bölümlerini (Domain, Configuration, Schema Partition) yapılandırdı.
- **Senaryo:** Şirkette bir "İnsan Kaynakları" klasörü (**Organizational Unit - OU**), bir de "Yazılım" klasörü açtın. Şema ise bir personelin hangi özelliklere sahip olabileceğini tanımlar (Örn: İsim, E-posta, IP Telefonu). Böylece sisteme *"Yazılımcılar sadece test laboratuvarındaki sunuculara uzaktan masaüstü (RDP) yapabilsin, muhasebeciler ise sadece finans klasörünü görsün"* kuralını bu mantıksal mimari sayesinde saniyeler içinde uygularsın.

**3. SYSVOL Paylaşımı — Şirket Hoparlörü ve Merkezi Emir Dağıtım Kanalı**

Normalde 25 bilgisayara bir ayar çekmek veya program yüklemek istesen, tek tek masaları gezip USB takman gerekirdi.

- **Ne oldu?** Sunucuda ağa açık, replike edilen `\\DOMAIN\SYSVOL` adında özel bir klasör oluştu. Burası Grup İlkelerinin (GPO) ve betiklerin (PowerShell/Batch script) barındığı yerdir.
- **Senaryo:** Şirket politikası gereği dedin ki: *"Pazartesi sabahı tüm bilgisayarların masaüstü arka planı şirket logosu olsun, USB bellek takmak yasaklansın ve açılışta herkese şirketin VPN sertifikasını yükleyen bir script çalışsın."* Sen bu kuralı sunucudaki Group Policy arayüzünden tanımlarsın; bu emirler SYSVOL klasörüne dosya olarak yazılır. 25 bilgisayar açıldığı anda arka planda bu paylaşıma bağlanıp emirleri okur ve sen yerinden bile kalkmadan 25 bilgisayarın ekranı aynı anda değişir, USB portları anında kilitlenir.

**4. DNS ve SRV Kayıtları — Şirketin Santrali ve Rehberi**

25 bilgisayar, bu sunucunun etki alanı denetleyicisi olduğunu nereden bilecek?

- **Ne oldu?** Sunucun kendi bünyesinde bir DNS kurdu ve içine `_ldap._tcp.dc._msdcs...` gibi özel **"Servis Kayıtları" (SRV Records)** yazdı.
- **Senaryo:** Ofisteki bilgisayarlardan biri domain'e katılacağı zaman ağa *"Ben bu şirketin sunucusunu arıyorum, nerede bu patron?"* diye bağırır. DNS sunucusu hemen cevap verir: *"Patron 10.0.2.10 IP adresinde, Kerberos doğrulaması için 88 portuna, LDAP sorgusu için 389 portuna git."* Eğer bu DNS kayıtları olmasaydı, 25 bilgisayarın hiçbiri sunucuyu bulamaz ve iletişim tamamen kopardı.

Kısacası Install tuşuna basarak, odaya bağımsız bir bilgisayar koymak yerine şirketin tüm kimliklerini, kurallarını, güvenlik politikalarını ve santralini tek bir merkezde toplayan **otorite merkezini** inşa etmiş oldun.

---

<a id="bolum-1-6"></a>
## 1.6. Kurulumu Doğrulama

Masaüstü açıldığında `Win + R` yapıp `cmd` yazarak komut istemcisini aç ve şu iki komutu çalıştır:

**Etki Alanı Kimliğini Görme:**

```cmd
whoami
```
Çıktıda selo\administrator görüyorsan makine artık yerel değil, doğrudan domain yöneticisi yetkisindedir.

###DNS ve DC İletişim Testi:
```cmd
nltest /dsgetdc:selo.local
```
(Eğer domain adını selo.local yaptıysan bu komut sana sunucunun kendi DC rolünü, IP adresini ve DNS SRV kayıtlarının çalıştığını listeler.)


<a id="bolum-1-7"></a>
## 1.7. ADUC ile Kullanıcı, Grup ve OU Yönetimi
Artık 25 kişilik şirketi yöneteceğin ana yönetim paneline geçebilirsin:

1- Açılan Server Manager penceresinde sağ üst köşedeki Tools (Araçlar) menüsüne tıkla.

2- Açılan listeden Active Directory Users and Computers (ADUC) seçeneğine tıkla.

3- Karşına gelen sol panelde selo.local ağacını genişlet. Burada Computers, Domain Controllers, Users gibi hazır klasörler göreceksin.

### Test Kullanıcısı ve Departman (OU) Oluşturma
Kullanıcıları doğrudan varsayılan Users klasörüne atmak kurumsal yapılarda tercih edilmez; departman bazlı düzen için OU (Organizational Unit) açılır:

### 1. Departman Klasörü Aç:

ADUC konsolunda selo.local domain adına sağ tıkla > New (Yeni) > Organizational Unit.

İsim olarak örneğin Bilgiİslem veya SiberLab yazıp OK de.

### 2. Kullanıcı Tanımla:

Oluşturduğun bu yeni OU'ya sağ tıkla > New > User.

First name: Ahmet, User logon name: ahmet yazıp Next de.

Parola belirle (örn: LabUser123!).

Alttaki "User must change password at next logon" seçeneğinin işaretini kaldırıp "Password never expires" kutusunu işaretle (lab ortamında parola sıfırlamayla uğraşmamak için), ardından Next > Finish de.

### 3. Güvenlik Grubu Aç:

Aynı OU'ya sağ tıkla > New > Group.

Grup adına SOC-Analistleri yaz, Group scope: Global, Group type: Security kalsın, OK de.

Ahmet kullanıcısına sağ tıkla > Add to a group > kutuya SOC-Analistleri yazıp Check Names ve ardından OK de.

<a id="bolum-1-8"></a>
## 1.8. İstemci Makineleri Domain'e Katma
İstemci (client) sanal makinelerin bu etki alanına katılabilmesi için 3 aşamalı bir kural zinciri bulunur:

### Adım A: Ağ Kartlarının Eşitlenmesi (VirtualBox)
Domain Controller makinen ile domain'e katacağın sanal istemci makinenin VirtualBox ayarlarındaki Ağ (Network) sekmesinde aynı modda olması şarttır.

Her iki makinenin de NAT Network (NAT Ağı) veya Host-only (Yalnızca Anamakine) modunda aynı ağ profiline bağlı olduğundan emin ol.

### Adım B: İstemcinin DNS Ayarı (En Kritik Kural)
İstemci bilgisayar, selo.local domain denetleyicisini bulabilmek için sorgularını senin DC'ye sormak zorundadır:

Domain'e katmak istediğin sanal Windows makinesini aç.

Win + R > ncpa.cpl yazıp Ethernet bağdaştırıcısına sağ tıkla > Properties > IPv4.

Preferred DNS server (Yeğlenen DNS sunucusu) kutusuna Domain Controller makinenin statik IP adresini (örn. 10.0.2.10) yaz ve kaydet.

Terminalden (cmd) ping selo.local yazarak alan adının DC IP'sine çözüldüğünü teyit et.

> **" Not: Kurumsal ağlarda bu ayar elle yapılmaz. DHCP sunucusu, IP dağıtırken DNS bilgisini de otomatik olarak istemciye basar. Lab ortamında elle yapıyoruz." **

## Adım C: Domain'e Katılma
İstemci Windows makinesinde Bu Bilgisayar (This PC) simgesine sağ tıkla > Özellikler (Properties) de.

Gelişmiş Sistem Ayarları > Bilgisayar Adı (Computer Name) sekmesine geçip alttaki Değiştir (Change) butonuna bas.

Alttaki "Üye olduğu yer" kısmında Workgroup yerine Domain (Etki Alanı) kutucuğunu işaretle.

Kutuya selo.local yazıp Tamam'a bas.

Senden yetkili kullanıcı adı ve şifre isteyecektir:

Kullanıcı: SELO\Administrator

Şifre: (Kurulumda belirlediğin Administrator parolası)

Ekrana "Welcome to the selo.local domain" penceresi geldiğinde makineyi yeniden başlat. Yeniden başladıktan sonra o makinede SELO\ahmet yazıp şifresini girerek oturum açabilirsin; sunucundaki ADUC panelinde Computers klasörüne baktığında istemci makinenin listeye eklendiğini göreceksin.

> **"Yasal Uyarı: Bu içerik yalnızca eğitim ve yetkili güvenlik testleri içindir. İzinsiz sistemlere erişim yasa dışıdır ve etik değildir."**

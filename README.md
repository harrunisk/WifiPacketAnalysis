# WifiPacketAnalysis

We can say that Wireshark is graphical version of Tshark. Aim of this article is to analyze some attributes of  wireless packets captured using tshark on Elasticsearch. Tshark, Elasticsearch, Kibana, Logstash and Filebeat are used to analyze.. Installation of Elasticsearch, Kibana, Logstash and Filebeat [on this link](https://www.elastic.co/products). An alternative solution is Docker. In a nutshell:
1.Tshark captures wireless packets by using filters.
2.Tshark writes captured wireless packets as .csv.
3.Filebeat listens .csv file sends to Logstash.


Tshark'ın Wireshark konsol arayüzü ya da Wireshark'ın tsharkın grafik arayüzü olan hali olduğu söylenebilir. 
Tshark ile yakalanan wireless paketlerinin bazı özelliklerinin analizini Elasticsearch ve araçları yardımı ile yapacağım.
Kullanacağım araçlar Tshark, ElasticSearch ,Kibana, Logstash ve Filebeat olacak. Elasticsearch, Kibana, Logstash ve Filebeat 
kurulumu için [bu link](https://www.elastic.co/products). Ya da docker kullanarak da halledebilirsiniz. Tshark filtreler kullanarak paketleri yakalayacak csv türünde 
dosyaya yazacak. Filebeat dosyayı dinleyecek Logstash'e aktaracak. Logstash burada yeniden filtreleme yapacak ve
Elasticsearch'e aktaracak. Kibana kullanarak analizlerimi yapacağım.
Mimarimiz aşağıdaki şekilde olacak:  
![Architecture](https://raw.githubusercontent.com/harrunisk/harrunisk.github.io/master/img/ArchitectureBlog.png)  
Wifi paketlerinin analizinde kullanacağım filtreler:  

Filtre   |  Açıklama
---------   |  ---------
_ws.col.Time   |  Zaman Bilgisi
wlan.fc.type   |   Çerçeve Türü
wlan.fc.type_subtype   |   Çerçeve Alt Türü
radiotap.dbm_antsignal   |   Sinyal Gücü(RSSI)
frame.len   |   Çerçeve Boyutu
radiotap.datarate   |   Veri Akış Hızı


wlan.fc.type filtresinden gelecek değerler 0, 1, ya da 2 olacak. Bu değerlerin karşılığı: 

Değer   |   Sonuç
-----   |   -----
wlan.fc.type==0   |   Yönetim Çerçevesi
wlan.fc.type==1   |   Kontrol Çerçevesi
wlan.fc.type==2   |   Veri Çerçevesi

wlan.fc.subtype filtresinden gelecek değerler 0 ile 47 arasında değişecek bunlardan bazıları aşağıdaki tabloda kapsamlı olarak [buradan](https://dalewifisec.wordpress.com/2014/04/29/wireshark-802-11-display-filters-2/) ulaşabilirsiniz.  

Değer   |   Sonuç
-----   |   -----
wlan.fc.type_subtype==8   |   Yönetim Çerçevesi
wlan.fc.type_subtype==27   |   Request To Send
 wlan.fc.type_subtype==28   |   Clear To Send



## 1.Paket Yakalama ve Dosyaya Yazma  
Kullanacağım filtre:
~~~
tshark -a duration:600 -i phy0.mon -t ad -t ad -lT fields -E separator=, -E quote=d   -e _ws.col.Time  -e wlan.fc.type -e wlan.fc.type_subtype -e radiotap.dbm_antsignal -e frame.len -e radiotap.datarate	 > tshark.csv
~~~
`-a duration:600`  10 dakika boyunca paket yakalayacağımı belirtiyorum.  
`-i phy0.mon` paketleri yakalayacak arayüzümü seçiyorum sizde farklılık gösterebilir.  
`-t ad` burada zamanın nasıl yazdırılacağını belirtiyorum. YYYY-MM-DD formatında.  
`> tshark.csv` yakalanan paketlerin yazılacağı dosya otomatik oluşturuyor.  
Kalan alanlar kullanılan filtrelerin ne şekilde ayrılacağı ile ilgili.  

## 2.Yakalan Paketlerin Filebeat İle Dinlenmesi
Filebeat 1.adımda oluşturduğumuz tshark.csv dosyasını dinleyecek ve buradaki değişiklikleri Logstash'e aktaracak. Bunun için filebeat.yml dosyasını aşağıdaki şekilde değiştiriyoruz.  
~~~
filebeat.modules:
- module: system
  syslog:
    enabled: false
  auth:
    enabled: true
    var.paths: ["/home/tshark.csv"]
name: test
output.logstash:
  hosts: ["localhost:5044"]
~~~
İndirme linki [filebeat.yml](https://raw.githubusercontent.com/harrunisk/WifiPacketAnalysis/master/filebeat.yml)  
`var.paths: ["/home/tshark.csv"]` dosyamızın yolu kendinize göre değiştirin.  
Windows sistemlerde filebeat'in kurulu olduğu yerden filebeat.yml dosyasına erişebilirsiniz sanırım.  
filebeat.yml dosyasına linux sistemler için aşağıdaki yol ile ulaşabilirsiniz:
~~~
cd /etc/filebeat
sudo subl filebeat.yml
~~~
## 3.Logstash'in Filebeat'i Dinlemesi Elastic Searche Yönlendirmesi ve CSV Dosyasına Filtrelerin Uygulanması

2.adımda Filebeat paketleri localhost:5044'e yani Logstash'in çalıştığı sokete yönlendirdi
Bu adımda gelen paketleri toplayıp ElasticSearch'e yönlendireceğiz. Logstash.yml dosyasında ilk olarak input kısmını yani filebeatden gelen paketleri topladığımız yeri değiştireceğiz:
~~~
input {
  beats {
    port => 5044
  }
}
~~~
Daha sonra ise output kısmını Elasticsearche'e yönlendireceğiz:  
~~~
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}" 
    document_type => "%{[@metadata][type]}" 
  }
}
~~~
Eper x-pack security gibi araçlar kullanıyorsanız output kısmına kullanıcı adı ve şifre girmeniz gerekecek:
~~~
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}" 
    document_type => "%{[@metadata][type]}" 
    user => elastic
    password => changeme
  }
}
~~~
En son olarakta verileri analiz edebilmek için filtre kısmını ayarlamamız gerekiyor burada filtreleri parça parça vereceğim bu adım sonunda da logstash.yml dosyasının tamamına ulaşabilirsiniz.  

Aşağıdaki filtre ile tshark'da oluşan alanların isimlerini belirliyoruz ve ayırıyoruz.
~~~
   csv {
     source => "message"
     columns => [ "col.time","frame.type","frame.subtype","rssi","frame.size","data.rate" ]
   }
~~~  
Aşağıdaki filtre ile tsharkın ürettiği zaman değerlerini Elasticsearch'de zamana göre analiz yapabilmek için eşitliyoruz:
 ~~~ 
date {
              match => [ "col.time", "YYYY-MM-DD HH:mm:ss.SSSSSSSSS" ]
              target => "@timestamp"
              }

~~~ 
Aşağıdaki filtre ile veri setimize saat, dakika ve saniye alanlarını ekliyoruz:
  ~~~ 
  mutate {
  add_field => {"[hour]" => "%{+HH}"}
  add_field => {"[minute]" => "%{+mm}"}
  add_field => {"[second]" => "%{+ss}"}

  }
  ~~~ 
Aşağıdaki filtre ile veri setimizdeki değerleri matematiksel işlemler yapabilmek için integer tipine parse ediyoruz.
~~~
mutate {
     convert => [ "rssi", "integer" ]
     convert => [ "frame.size", "integer" ]
     convert => [ "data.rate", "integer" ]   
     convert => [ "second", "integer" ]  
     convert => [ "minute", "integer" ]  
     convert => [ "hour", "integer" ]  

   }
~~~

Aşağıdaki filtre ile çerçevelerin matematiksel karşılıklarını anlamlı veriler haline getiriyoruz.
~~~
   if[frame.type]=="0"{
   mutate {
     replace => [ "frame.type", "Management" ]
   }}
   if[frame.type]=="1"{
   mutate {
     replace => [ "frame.type", "Control" ]
   }}
   if[frame.type]=="2"{
   mutate {
     replace => [ "frame.type", "Data" ]
   }}
~~~  
  
Logstash.yml dosyasının son hali şu şekilde olmalı: [Logstash.yml](https://raw.githubusercontent.com/harrunisk/WifiPacketAnalysis/master/logstash.conf).   
#### Düzeltme  
Eğer yukarıdaki gibi Logstash.yml ile çalışmıyorsa. Logstash.conf isimli bir konfigürasyon dosyası oluşturup yukarıdaki Logstash.yml dosyasını içine kopyalayın. Ve logstash dosyalarının kurulu olduğu yerde bin klasörü içinde komut satırından:
~~~
./logstash -f Logstash.conf 
~~~
ile çalıştırın -f hangi konfigürasyon dosyası ile çalışacağını belirtiyor. Burada dosyayı kaydettiğiniz yolu da girmeniz gerekiyor.
## 4.Son Adım
Özet olarak yukarıdaki konfigürasyonları yaptıktan sonra tshark ile paketleri topluyoruz.
~~~
tshark -a duration:600 -i phy0.mon -t ad -t ad -lT fields -E separator=, -E quote=d   -e _ws.col.Time  -e wlan.fc.type -e wlan.fc.type_subtype -e radiotap.dbm_antsignal -e frame.len -e radiotap.datarate	 > tshark.csv
~~~
Elasticsearch, Kibana, Logstash ve Filebeat'i aktif hale getiriyoruz.  
Yükleme tarzınıza göre değişmekle birlikte linux sistemler için:
~~~
sudo systemctl start elasticsearch.service kibana.service logstash.service filebeat.service  
~~~

Belirli bir süre geçtikten sonra verileri Elasticsearch içinde indekslemiş olacak. Yeni bir index pattern oluşturarak. `Management>Index Patterns >Create Index` diyerek yeni bir index pattern oluşturduktan sonra `Visualize` sekmesinden kendinize göre analizler yapabilirsiniz. Örnek analizler:  

![Physical Layer](https://raw.githubusercontent.com/harrunisk/harrunisk.github.io/master/img/physicalLayerPacketSize.png)    

![Pie](https://raw.githubusercontent.com/harrunisk/harrunisk.github.io/master/img/pieFrameType.png) 

![DataRate](https://raw.githubusercontent.com/harrunisk/harrunisk.github.io/master/img/dataRate.png) 

  
### Kaynaklar  
    
  
[Tshark komutları](https://www.wireshark.org/docs/man-pages/tshark.html)  

[Wlan filtreleri](https://www.wireshark.org/docs/dfref/w/wlan.html) 

[https://rudibroekhuizen.wordpress.com/2016/02/12/analyse-tshark-capture-in-kibana/](https://rudibroekhuizen.wordpress.com/2016/02/12/analyse-tshark-capture-in-kibana/)  

[http://www.lovemytool.com/blog/2010/02/wireshark-wireless-display-and-capture-filters-samples-by-joke-snelders.html](http://www.lovemytool.com/blog/2010/02/wireshark-wireless-display-and-capture-filters-samples-by-joke-snelders.html)  

[https://dalewifisec.wordpress.com/2014/04/29/wireshark-802-11-display-filters-2/](https://dalewifisec.wordpress.com/2014/04/29/wireshark-802-11-display-filters-2/)























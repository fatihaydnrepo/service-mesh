# Service Mesh :globe_with_meridians: :syringe:

<p>

Mikroservis mimarisinin hayatımıza girmesiyle birlikte büyük çaplı uygulamaların yönetiminde bazı problemler ortaya çıkmaya başladı. Bu problemlerin küçük uygulamalarda izlenebilirlik kolaylığı sebebiyle çözümü çok fazla efor istemese de büyük ve çok sayıda mikroservisten oluşan uygulamalarda izlenebilirlik zor olduğu için çok fazla efor istemektedir. Örnek olarak 80 mikroservisten oluşan bir uygulamamızdaki trafiğin nasıl aktığını ve eğer bu trafikte bir kayıp olursa bunun hangi iki servis arasında yaşandığını çözümlemek oldukça zor olacaktır. Tam olarak bu noktada bizlere önemli bir çözüm sunan **service mesh**, uygulamamızdaki mikroservislerin yanında bir sidecar proxy olarak çalışarak trafiği üstlenir ve izlenebilirliği arttırır. **Service mesh** sayesinde bütün trafiği izleyebilir, trafiği yönetebilir, trafik yönetme sayesinde blue-green deployment yapabilir ve mikroservisler arasındaki trafiği virtual TLS ile şifreleyebiliriz. 

<p/>
<p>

Kubernetes'in kendisi bu gibi durumlar için bizlere bir çözüm sağlamamakta anca biz daha sonrasından cluster bazında tüm trafiğimizi kontrol etmek ve yönetmek için farklı araçları cluster üzerine entegre ederek kullanabiliriz.

<p/>

<p>

Özetle **service mesh** bizlere aşağıdaki konularda çözümler sunar:
- Routing & Load Balancing
- Canary Deployment
- Dağıtılmış izleme yoluyla gözlemlenebilirlik
- mutual Transport Level Security (mTLS)
- "Service-to-Service" trafiği loglarına erişim

<p/>

## Yaygın Service Mesh Araçları
### Istio
<p> 

![istio](https://devopscube.com/wp-content/uploads/2021/09/arch.png/)

<p/>

### Linkerd
<p>

![linkerd](https://devopscube.com/wp-content/uploads/2021/09/control-plane.png/)

<p/>

### Consul Connect
<p>


![consul](https://cdn.haproxy.com/documentation/hapee/latest/assets/service-mesh-consul-diagram-f862a2ea815a9938148317b26465583471149d526f975b48596af7ebda1f795e.png)

<p/>


### Traefik
<p>

![traefik](https://devopscube.com/wp-content/uploads/2021/09/before-traefik-mesh-graphic.png/)

<p/>


### Open Service Mesh (OSM)
<p>

![osm](https://devopscube.com/wp-content/uploads/2021/09/osm-components-and-interactions.png/)

<p/>


### Nginx Service Mesh
<p>

![nginx](https://devopscube.com/wp-content/uploads/2021/09/NGINX-Service-Mesh-sol-brief-2021_featured.png/)

<p/>


### Kuma
<p>

![kuma](https://konghq.com/wp-content/uploads/2020/06/Kuma-0.6-Diagram-1024x707.jpg)

<p/>


##  Linkerd :syringe:

<p>

Linkerd, service mesh teknolojisinde bizlere çözüm sunan bir araçtır. CNCF tarafından yönetilmektedir. Çalışma prensibi olarak Linkerd bir API kurmaktadır ve bütün komponentler bu API ile haberleşir. Ek olarak linkerd dahsboard ve CLI da bu API ile haberleşerek bizler için gerekli bilgileri sağlar. Linkerd genel olarak iki ana başlıktaki yapıya bölünmektedir. Bunlardan ilki linkerd'nin çalışmasında core görevi gören **Control Plane**'dir. Diğeri ise uygulamalarımızın yanında sidecar proxy olarak çalışan **Data Plane** dir.

<p/>

<p>

Ek olarak linkerd ve Istio OSI Reference Model Layer 7(Application) çalışır. HTTP veya GRPC protokolleri kullanır.

<p/>

<p>

![plane](https://miro.medium.com/max/1400/0*ovTtAG8H368CKh0i)

<p/>


#####  Control Plane
<p>

Bu katmanda üç adet araç bulunur. Bunlar "destination", "identity" ve "proxy-injector" dür. 

<p/>

<p>

Destination: Görevi A servisi ile B servisi arasındaki trafiği yöntemek veya sonrasında trafiğin nereye gideceğini belirlemektir. Genel olarak trafiğin yönlendirilmesinden sorumludur.

<p/>

<p>

Identity: Virtual TLS altyapısını sağlayabilmek için gerekli sertifikaları podlara inject eder. Yeni bir servis oluşuturup bu servisi linkerd altına aldığımızda da bu servisin sertifika işlemleri Identity tarafından sağlanır.

<p/>

<p>

Proxy-Injector: Çalışan bir cluster için öncelikle servislerimizin yanlarına sidecar olarak linkerd proxy kurmamız gerekmektedir. Proxy-injector ise burada bu sidecar proxylerin kurulumu ve yönetiminden sorumludur.

<p/>

#####  Data Plane
<p>

Bu katman linkerd'nin uygulama tarafında çalışan kısmıdır. İçerisinde linkerd aracı olarak sadece sidecar proxy bulunur. Her servise entegre olarak bu sayede bütün trafiği üstlenir ve izleyip yönetmemize olanak sağlar.

Data Plane uygulamamıza infrastructure seviyesinde dahil olur.

<p/>





## Linkerd Kurulumu
<p>

Anlaşılması ve basitliği sebebiyle **Linkerd** üzerinden bir kurulum gerçekleştirip ardından bir tane de örnek uygulama kurarak uygulama özelindeki trafiği inceleyeceğiz. 

<p/> 
<p> 

Ön Gereksinimler:

* Docker Desktop (İsteğe Bağlı)
* Kubernetes Cluster 
* Lens (İsteğe Bağlı)

<p/>

Bu aşamada k8s clusterı docker desktop ile ayağa kaldırdık. Farklı bir ortamdaki cluster da kullanılabilir. Ek olarak Lens uygulamasını ile cluster bilgilerini görüntülemek için kullandık. `kubectl` cli kullanılarak da gerekli kontroller sağlanabilir. <p/>

##### 1. K8s cluster erişim kontorlü
<p>

    kubectl version --short
Öncelikle bu komut ile mevcut clustera erişimimiz olup olmadığının kontrolünü sağlıyoruz.
<p/>

#####  2. Linkerd CLI kurulumu
<p>

    curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
Bu komutu shell üzerinde çalıştırarak bilgisayarımıza linkerd cli kurulumunu yapıyoruz. Ardından
```
 export PATH=$PATH:/home/nadi/.linkerd2/bin
```
  
    linkerd version
   sırasıyla komutları çalıştırarak kurulumun gerçekleştiğini doğruluyoruz.
   
<p/>

#####  3. K8s cluster için Linkerd kontrolü
<p>

    linkerd check --pre
Bu adımda clusterımız için linkerd gereksinimlerinin kontrollerini sağlıyoruz. Eğer bir sorun ile karşılaşılmazsa komut sonucunda 'Status check results are ok' şeklinde bir çıktı alıyoruz.
<p/>

#####  4. Linkerd Control Plane kurulumu
<p>

    linkerd install | kubectl apply -f -
Buradaki "linkerd install" komutu control plane için gerekli manifest dosyalarını oluşturur. Sonrasında ise "kubectl apply -f -" ile ilgili dosyalar cluster içerisine kurulur.
<p/>

#####  5. Control Plane kurulumu kontrolü
<p>

    linkerd check
Control plane kurulumu aşamasında bir hata ile karşılaşılmadı ve control plane clustera doğru bir şekilde kurulduysa komut sonucunda "Status check results are ok" şeklinde bir çıktı alıyoruz.
<p/>

#####  6. Demo uygulamanın yüklenmesi
<p>

    curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/emojivoto.yml | kubectl apply -f -
Bu adımda emojivoto isimli ve içerisinde hata bulunan bir uygulamayı deploy ediyoruz. İçerisindeki hata bilinçi olarak oluşturulmuş, uygulamada trafik kaybına neden olan bir hatadır.

<p/>

<p>

**NOT:** Linkerd şu anda uygulamaya inject:syringe: edilmemiştir, yani çalışmamaktadır.

<p/>

#####  7. Uygulamanın çalıştığının kontrolü
<p>

    kubectl -n emojivoto port-forward svc/web-svc 8080:80
Kubernetes port-forward ile uygulamamızın web servisine http://localhost:8080 adresinden erişim sağlayalım. Uygulamamız basit bir şekilde emoji oylama uygulamasıdır. Açılan ekranda rastgele oylama yaparak uygulamayı kontrol edebiliriz. Donut emojisine tıkladığımızda hata aldığımızı görüntüleyeceksiniz. Bu kısım uygulamanın bilinçli olarak hatalı oluşturulan kısmıdır.

<p/>

<p>

**NOT:** Linkerd şu anda hala uygulamaya inject:syringe: edilmemiştir, yani çalışmamaktadır.

<p/>

#####  8. Linkerd' yi uygulamaya inject:syringe: etme
<p>

    kubectl get -n emojivoto deploy -o yaml | linkerd inject - | kubectl apply -f -
Bu komut satırında sırası ile "emojivoto" namespace içerisindeki deployment objelerinin yaml dosyalarını alıyoruz. Ardından **linkerd inject** komutu ile uygulamamıza linkerd data plane ekliyoruz, yani uygulamıza linkerd ekliyoruz. Son olarak ise aynı dosyaları tekrar deploy ediyoruz.
<p/>

#####  9. Data Plane kontrolü
<p>

    linkerd -n emojivoto check --proxy
Bu aşamada oluşturduğumuz data plane çalışıyor mu kontrolünü sağlıyoruz. Eğer bir hata ile karşılaşılmamış ise sonucunca "Status check results are ok" çıktısını elde ediyoruz.
<p/>

#####  10. Linkerd dasboard için "Viz" eklentisi kurulumu
<p>

    linkerd viz install | kubectl apply -f -
Sırası ile bu komutumuzda viz eklentisinin gerekli manifest dosyalarını oluşturup ardından clustera deploy ediyoruz. 

<p/>

<p>

**NOT:** Viz eklentisi beraberinde monitoring için prometheus ve grafana da kuruyor. Bu nedenle mevcut prometheus kurulumunuz var ise çakışma veya başka bir problem olmaması için kaldırmanız gerekiyor.

<p/>

#####  11. Viz eklentisi kontrolü
<p>

    linkerd check
Bu komut ile yüklediğimiz eklenti çalışıyor mu  ve eklenti kurulduktan sonra linkerd komponentleri düzgün çalışıyor mu bunların kontrollerini sağlıyoruz. Eğer bir hata ile karşılaşılmamış ise sonucunca "Status check results are ok" çıktısını elde ediyoruz.
<p/>

#####  12. Viz eklentisi çalıştırma
<p>

        linkerd viz dashboard &
Komutu bizleri  http://localhost:50750 adresine yönlendiriyor ve karşımıza linkerd dahboard ekranı geliyor.
Ek olarak grafa üzerinde oluşturulmuş default dasboard için http://localhost:50750/grafana bağlantısını kullanabiliriz.
<p/>


<p>

![viz](https://linkerd.io/images/getting-started/viz-empty-dashboard.png)

<p/>

#####  13. Mimari :wrench:
<p>

![mimari](https://i.ibb.co/jW0t3ST/linkerd.jpg/)

<p/>


<p>

Dört mikroservisten oluşan uygulamamızdaki "Voting" servisi içersinde **donut** APİ hatalı olduğu için bu kısımda bağlantı kaybı oluşmaktadır. Bu kayıp üstteki resimde başarı oranı altında gözlenmektedir. Aynı zamanda web servisi ile arasındaki trafiği de bozduğu için web servisinde de bağlantı kaybı görüntülenmesine neden olmaktadır.

<p/>

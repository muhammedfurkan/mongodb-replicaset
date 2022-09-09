# MongoDB ReplicaSet Ayarlaması

3 tane sunucumuz olduğunu varsayalım. Bunlar; **mongo0**, **mongo1 ve** **mongo2** olsun.

## 1. Adım - DNS Yapılandırması

Serverler için DNS yapılandırması gerekir. Bunu yapabilmek için `/etc/host` dizinindeki ayarlamalar aracılığıyla yapacağız.

```bash
sudo nano /etc/hosts
```

Her satırda bir server IP ve Hostname eklememiz gerekir. Örnek;

```bash
203.0.113.0 mongo0.replset.member
203.0.113.1 mongo1.replset.member
203.0.113.2 mongo2.replset.member
```

## 2. Adım - UFW Kullanarak Güvenlik Duvarı Ayarlarının Yapılması

Aşağıdaki ayarlamaları her bir sunucu için ayrı ayrı yapılması gerekmektedir. 

```bash
sudo ufw allow from mongo1_server_ip to any port 27017
```

## 3. Adım - Her Bir Sunucu İçin MongoDB Yapılandırma

Bir önceki adımlarda `/etc/hosts` yapılandırmasını yapmıştık. Daha sonra yine her bir sunucu için güvenlik duvarı yapılandırmasını ayarladık ve varsayılan port olan 27017 portunu kullandık.

Bu adımda `/etc/mongod.conf` dosyasında düzenlemeler yapacağız.

**Mongo0** sunucusunda aşağıdaki komutu kullanıp ayarlamalara başlayalım.

```bash
sudo nano /etc/mongod.conf
```

Diğer sunucuların 27017 numaralı portuna erişmeye izin vermek için her sunucunun güvenlik duvarını açmış olsanız bile, MongoDB şu anda yerel geri döngü ağ arabirimi olan 127.0.0.1'e bağlıdır.

Uzak bağlantılara izin vermek için 127.0.0.1'e ek olarak MongoDB'yi sunucularınızın genel olarak yönlendirilebilir IP adreslerine bağlamanız gerekir. Bu sayede MongoDB kurulumunuz, uzak makinelerden MongoDB sunucunuza yapılan bağlantıları dinleyebilecektir.

Ağ arayüzleri bölümünü bulun. Varsayılan olarak şöyle görünecektir:

```bash
. . .
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1
. . .
```

**BindIp**: ile başlayan satıra bir virgül ekleyin: ardından **mongo0**'ın ana bilgisayar adı veya genel IP adresi.

```bash
. . .
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,**mongo0.replset.member**
. . .
```

Ardından, dosyanın altına doğru #replication: yazan satırı bulun. Bunun gibi görünür:

```bash
. . .
#replication:
. . .
```

(#) işaretini kaldırın. Ardından, bu satırın altına bir **replSetName** yönergesi ve ardından MongoDB'nin çoğaltma kümesini tanımlamak için kullanacağı bir ad ekleyin:

```bash
. . .
replication:
  replSetName: "rs0"
. . .
```

Bu örnekte **replSetName** yönergesinin değeri "**rs0**"dır. Burada istediğiniz adı verebilirsiniz, ancak açıklayıcı bir ad kullanmanız yararlı olabilir.

Yine de, her bir sunucunun **mongod.conf** dosyasının, MongoDB örneklerinin her birinin aynı replika kümesinin üyesi olabilmesi için **replSetName** yönergesinden sonra aynı ada sahip olması gerektiğini unutmayın.

**replSetName** yönergesinden önce iki boşluk olduğuna ve adın tırnak işaretleri ("") içine alındığını ve her ikisinin de bu yapılandırmanın düzgün bir şekilde okunması için gerekli olduğunu unutmayın.

Tüm güncellemeler sonrasında `/etc/mongod.conf` bu şekilde görünecektir: 

```bash
. . .
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,mongo0.replset.member
. . .
replication:
  replSetName: "rs0"
. . .
```

```bash
. . .
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,mongo1.replset.member
. . .
replication:
  replSetName: "rs0"
. . .
```

```bash
. . .
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,mongo2.replset.member
. . .
replication:
  replSetName: "rs0"
. . .
```

**Yinelemek gerekirse, her sunucunun bindIp yönergesine eklediğiniz IP adresi veya ana bilgisayar adı, mongod.conf dosyasını düzenlediğiniz sunucunun adresi olmalıdır.**

Her sunucunun mongod.conf dosyasında bu değişiklikleri yaptıktan sonra, her dosyayı kaydedin ve kapatın. Ardından, aşağıdaki komutu vererek her sunucuda mongod hizmetini yeniden başlatın:

```bash
sudo systemctl restart mongod
```

## 4.Adım - Replica Başlatma ve Üye Ekleme

**Bu örnekte mongo0 sunucusuna gelen datalar mongo1 ve mongo2 sunucusunda bulunan mongoya replica olacaktır.**

mongo0 sunucusunda komut satırını açıp aşağıdaki komutu çalıştıralım;

```bash
mongo
```

Komut isteminden, **rs.initiate()** yöntemini çalıştırarak mongo kabuğundan bir çoğaltma kümesi başlatabilirsiniz. Ancak, bu yöntemi kendi başına çalıştırmak, yalnızca yöntemi çalıştırdığınız makine için çoğaltmayı başlatır ve daha sonra her üye için bir **rs.add()** yöntemi yayınlayarak diğer Mongo örneklerinizi eklemeniz gerekir.

```bash
> rs.initiate(
... {
... _id: "rs0",
... members: [
... { _id: 0, host: "mongo0.replset.member" },
... { _id: 1, host: "mongo1.replset.member" },
... { _id: 2, host: "mongo2.replset.member" }
... ]
... })
```

Tüm detayları doğru girdiğinizi varsayarsak, kapanış parantezini yazdıktan sonra ENTER'a bastığınızda metot çalışacak ve replika setini başlatacaktır. Yöntem çıktıda **"ok" : 1** döndürürse, çoğaltma kümesinin doğru başlatıldığı anlamına gelir:

```bash
{
    "ok" : 1,
    "$clusterTime" : {
        "clusterTime" : Timestamp(1612389071, 1),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    },
    "operationTime" : Timestamp(1612389071, 1)
}
```

Replica kümesi beklendiği gibi başlatıldıysa, MongoDB istemcisinin isteminin yalnızca büyüktür işaretinden (>) aşağıdakine değişeceğini fark edeceksiniz:

```bash
rs0:SECONDARY>
```

rs komutlarından herhangi biri çalıştırıldığında terminal çıktısı aşağıdaki gibi olabilir. Bu, bağlı olduğunuz MongoDB örneğinin birincil küme üyesi olarak hizmet vermek üzere seçildiği anlamına gelir.

```bash
rs0:PRIMARY>
```

İleride replica sunucu eklemek istediğiniz ek sunucular varsa, bunları önceki adımlarda geçerli replica sunucu üyelerinde yaptığınız gibi yapılandırdıktan sonra **rs.add()** yöntemiyle yapabileceğinizi unutmayın.

```bash
rs0:PRIMARY> rs.add( "mongo3.replset.member" )
```

Artık CTRL + C tuşlarına basarak veya çıkış komutunu çalıştırarak MongoDB terminalini kapatabilirsiniz.

## Sonuç

Artık mongo0 sunucusuna gelen datalar başarılı bir şekilde mongo1 ve mongo2 sunucusuna gitmekte.
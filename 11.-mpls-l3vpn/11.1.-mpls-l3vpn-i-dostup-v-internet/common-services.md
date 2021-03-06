# Common Services

До сих пор мы обсуждали задачу передачи трафика из VPN в публичные сети и обратно.  
Ещё один подход к предоставлению доступа в Интернет — вывести его в отдельный VRF.  
Строго говоря — он наиболее масштабируемый, потому что нет необходимости настраивать что-то индивидуально для каждого клиентского VRF.  
Важное условие — NAT происходит в сети клиента и в VRF Internet импортируются только маршруты к публичным префиксам клиента.

Common Services, который часто называют Shared Services — это продолжение вопроса о взаимодействии между VRF, который мы рассмотрели [ранее](https://habrahabr.ru/post/273679/#VRF_INTERCONNECTION). Основная идея в том, что экспорт/импорт совершается засчёт особой настройки route-target. Только на этот раз нужно ограничить список передаваемых префиксов, чтобы избежать пересечения адресных пространств и разрешить только публичные маршруты.

Рассмотрим задачу снова на примере TARS, потому что у них уже есть публичные сети.

На шлюзе мы создаём VRF Internet.

```text
Linkmeup_R1(config)#ip vrf Internet
Linkmeup_R1(config-vrf)#rd 64500:1
Linkmeup_R1(config-vrf)#route-target import 64500:11
Linkmeup_R1(config-vrf)#route-target export 64500:22
```

Обратите внимание, что route-target на импорт и на экспорт в этот раз разные и сейчас станет понятно почему.  
2\) Перенести интерфейс в сторону Интернета в VRF:

```text
Linkmeup_R1(config)#interface FastEthernet 1/1
Linkmeup_R1(config-if)#ip vrf forwarding Internet
Linkmeup_R1(config-if)#ip address 101.0.0.2 255.255.255.252
```

3\) Перенести BGP-соседство с маршрутизатором в Интернете в address-family ipv4 vrf Internet:

```text
Linkmeup_R1(config-router)#router bgp 64500
Linkmeup_R1(config-router)#no neighbor 101.0.0.1 remote-as 64501
Linkmeup_R1(config-router)#address-family ipv4 vrf Internet
Linkmeup_R1(config-router-af)#neighbor 101.0.0.1 remote-as 64501
Linkmeup_R1(config-router-af)#neighbor 101.0.0.1 activate
```

4\) Объявляем маршрут по умолчанию:

```text
Linkmeup_R1(config)#router bgp 64500
Linkmeup_R1(config-router-af)#network 0.0.0.0 mask 00.0.0.0
Linkmeup_R1(config-router-af)#default-information originate
```

```text
Linkmeup_R1(config)#ip route vrf Internet 0.0.0.0 0.0.0.0 101.0.0.1
```

5\) В клиентском VRF TARS нужно также настроить RT:

```text
Linkmeup_R1(config-vrf)#ip vrf TARS
Linkmeup_R1(config-vrf)# route-target both 64500:200
Linkmeup_R1(config-vrf)# route-target export 64500:11
Linkmeup_R1(config-vrf)# route-target import 64500:22
```

Итак, помимо собственного RT, который обеспечивает обмен маршрутной информацией с другими филиалами \(64500:200\), здесь настроены и те RT, которые и для VRF Internet, но наоборот:

* то, что было на экспорт в VRF Internet \(64500:22\), то стало на импорт в VRF TARS
* то, что было на импорт в VRF Internet \(64500:11\), то стало на экспорт в VRF TARS

Почему так? Почему нельзя просто дать route-target both 64500:1, например, на всех VRF?  
Основная идея концепции Common Service — предоставить доступ к нужному VRF клиентам, но не позволить им общаться друг с другом напрямую, чтобы обеспечить изоляцию, как того требует определение VPN.  
Если настроить одинаковый RT на всех VRF, то маршруты будут спокойно ходить между ними.  
При указанной же выше конфигурации у всех клиентских VRF есть RT 64500:22 на импорт \(все будут получать маршруты Internet\), и также у них есть RT 64500:11 на экспорт, но только у VRF Internet есть такой RT 64500:11 на импорт — только VRF Internet будет получать маршруты клиентов. Друг с другом они обмениваться не смогут. Главное, чтобы Internet не занялся филантропией и не начал маршрутизировать клиентский трафик.

![](https://habrastorage.org/getpro/habr/post_images/c61/9ad/97f/c619ad97f0ec6636e8bc1c3c75ec80cf.png)

Итак, в результате наших операций мы можем видеть следующее:

![](https://habrastorage.org/getpro/habr/post_images/487/7c2/a4f/4877c2a4f1cced935eca18389ffc88fe.png)

На TARS\_2 всё в порядке.

На Linkmeup\_R1 есть маршрут до сети 100.0.0.0/30, но есть и лишние маршруты до частных сетей:

![](https://habrastorage.org/getpro/habr/post_images/a10/430/415/a10430415d1e67b81e0f645ab773398c.png)

И в этом случае у нас даже будет интимная связность:

![](https://habrastorage.org/getpro/habr/post_images/50a/b2a/576/50ab2a576330e527931e4042dd45515a.png)

Но что делать с этими лишними маршрутами в VRF Internet? Ведь если мы подключим ещё один VRF так же, у нас и от него появятся ненужные серые подсети.

Тут как обычно поможет фильтрация. А если конкретно, то воспользуемся prefix-list + route-map:

```text
Linkmeup_R3(config)#ip prefix-list 1 deny 172.16.0.0/12 le 32
Linkmeup_R3(config)#ip prefix-list 1 deny 192.168.0.0/16 le 32
Linkmeup_R3(config)#ip prefix-list 1 deny 10.0.0.0/8 le 32
Linkmeup_R3(config)#ip prefix-list 1 permit 0.0.0.0/0 le 32
```

Первые три строки запрещают анонсы всех частных сетей. Четвёртая разрешает все остальные.  
В нашем случае вполне можно было бы обойтись одной строкой: **ip prefix-list 1 permit 100.0.0.0/23 le 32** — вся подсеть Linkmeup, но приведённый нами пример более универсальный — он допускает существование других публичных сетей и соответственно один prefix-list может быть применён для всех VRF.  
Следующая конструкция применяет расширенное community к тем префиксам, что попали в prefix-list 1, иными словами устанавливает RT:

```text
Linkmeup_R3(config-map)#route-map To_Internet permit 10
Linkmeup_R3(config-map)#match ip address prefix-list 1
Linkmeup_R3(config-map)#set extcommunity rt 64500:11
```

Осталось дело за малым — применить route-map к VRF:

```text
Linkmeup_R3(config)#ip vrf TARS
Linkmeup_R3(config-vrf)#export map To_Internet
```

После обновления маршрутов BGP \(можно форсировать командой **clear ip bgp all 64500**\) видим, что в VRF Internet остался только публичный маршрут:

![](https://habrastorage.org/getpro/habr/post_images/961/de1/431/961de143107c0d8a46ada0194df08b27.png)

И, собственно, проверка доступности Интернета с PC1 \(NAT уже настроен на TARS\_2\):

![](https://habrastorage.org/getpro/habr/post_images/f69/34a/194/f6934a1944842188f3b3b6511f6580df.png)

Уважаемые читатели, вы только что ознакомились с другим подходом к **Route Leaking**'у.

[Полная конфигурация всех узлов для Common Services.](https://docs.google.com/document/d/1LgWutlcWs30v6gs3z_17cDq7ZJEckGPFT6ABJaHTpCc/pub)

Наиболее доступно тема Common Services описана [Jeremy Stretch](http://packetlife.net/blog/2011/may/19/mpls-vpn-common-services/). Но у него нет указания на то, что префиксы нужно фильтровать.  
Вообще, у них там в ихних америках, все друг друга знают и уважают. Поэтому Джереми охотно ссылается на Ивана Пепельняка, а точнее его [заметку о Common Services](http://blog.ipspace.net/2011/05/mplsvpn-common-services-design.html), а Иван в свою очередь на Джереми. Статьи их дополняют друг друга, но опять же до конца тему не раскрывают.  
А вот и [третья ссылка](https://supportforums.cisco.com/discussion/11567421/route-leaking-between-vrfs-shared-services), которая в купе с первыми двумя позволяет сложить какое-то представление о том, как работает Common Services.


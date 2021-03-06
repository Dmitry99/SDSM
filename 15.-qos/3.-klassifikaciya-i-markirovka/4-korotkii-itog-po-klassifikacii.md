# Короткий итог по классификации

На входе в узел пакет классифицируется на основе интерфейса, MF или его маркировки \(BA\).

Маркировка — это значение полей DSCP в IPv4, Traffic Class в IPv6 и в MPLS или 802.1p в 802.1q.  
Выделяют 8 классов сервиса, которые агрегируют в себе различные категории трафика. Каждому классу назначается свой PHB, удовлетворяющий требованиям класса.  
Согласно рекомендациям IETF, выделяются следующие классы сервисов, это CS1, CS0, AF11, AF12, AF13, AF21, CS2, AF22, AF23, CS3, AF31, AF32, AF33, CS4, AF41, AF42, AF43, CS5, EF, CS6, CS7 в порядке возрастания важности трафика.  
Из них можно выбрать комбинацию из 8, которые реально можно закодировать в поля CoS.  
Наиболее распространённая комбинация: CS0, AF1, AF2, AF3, AF4, EF, CS6, CS7 с 3 градациями цвета для AF.  
Каждому классу ставится в соответствие PHB, которых существует 3 — Default Forwarding, Assured Forwarding, Expedited Forwarding в порядке возрастания строгости. Немного в стороне стоит PHB Class Selector. Каждый PHB может варьироваться параметрами инструментов, но об этом [дальше](https://github.com/eucariot/SDSM/tree/ef6db47d24b261587c7463bea30d1e7c7b6ece89/15.-qos/5.-ocheredi.md).

В незагруженной сети QoS не нужен, говорили они. Любые вопросы QoS решаются расширением линков, говорили они. С Ethernet и DWDM нам никогда не грозят перегрузки линий, говорили они.

Они — те, кто не понимает, что такое QoS.  
Но реальность бьёт VPN-ом по РКНу.  
1\) Не везде есть оптика. РРЛ — наша реальность. Иногда в момент аварии \(да и не только\) в узкий радио-линк хочет пролезть весь трафик сети.  
2\) Всплески трафика — наша реальность. Кратковременные всплески трафика легко забивают очереди, заставляя отбрасывать очень нужные пакеты.  
3\) Телефония, ВКС, онлайн-игры — наша реальность. Если очередь хоть на сколько-то занята, задержки начинают плясать.

> В моей практике были примеры, когда телефония превращалась в азбуку морзе на сети, загруженной не более чем на 40%. Всего лишь перемаркировка её в EF решила проблема сиюминутно.

Пришло время разобраться с инструментами, которые позволяют обеспечить разный сервис разным классам.


### Rocket Beans - API,  OAuth & Authorization

#### Umsetzung
Implementierung gem�� RFC 6749;
derzeit unterst�tzte Authorization Grants: 
	- Authorization Code
	- Implicit
	
wobei Authorization Code zu bevorzugen ist.


#### Endpunkte
 - Authorization Endpunkt:
 https://rocketbeans.tv/oauth2/authorize
	
- Token Endpunkt:
 https://api.rocketbeans.tv/v1/oauth2/token
		
####  Authentifizierung
Authentifizierung gegen�ber der API erfolgt �ber den Authorization Header mit bearer Prefix.

Beispiel Header:
``` 
Authorization: bearer ADFm+0C5XVDiyxooQKB-LRphiQzR18MoEA5L+.WTpRr5YOqt4Y-w-ZMgLGrA5anqMBhfvii5SBjPsMry.wTU6fBOn_90f-lmc
```



#### Informationen zu Access und Refresh Token
- Access Token
Lebensdauer: 	Bis zu 8 Stunden
	
- Refresh Token:
Lebensdauer:	Unbegrenzt bei regelm��iger Verwendung (token renew), 
			werden jedoch nach 3 monatiger Nichtverwendung eingezogen / ung�ltig.

Die genaue Time-to-Live wird bei der Herausgabe der Token �bermittelt.

Zum speichern der Token sollten bis zu 160 Zeichen je Token vorgesehen werden. 
Die verwendeten Zeichen entsprechen ASCII-7 (A-Za-z0-9+-_.)
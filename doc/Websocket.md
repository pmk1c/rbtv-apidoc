## Rocket Beans TV Websocket API v1

### Grunds�tzliches
Die Websocket Schnittstelle basiert auf Socket.IO 2.0.<br />
Die aktuelle URL sowie der Pfad wird �ber den Endpunkt /frontend/init zur Verf�gung gestellt (siehe *frontendInitResponse::websocket // interface websocketParameters*)<br />
*Siehe: (https://github.com/rocketbeans/rbtv-apidoc#frontend-initialization-information)*

Aus Kompatibilit�tsgr�nden sowie aus Gr�nden der Interaktivit�t (Latenz) sollte nur der websocket Transport verwendet werden.

Javascript / Socket.IO-client Beispiel:
```javascript
const URL = <websocketParameters.url>; // See endpoint: /frontend/init
const PATH = <websocketParameters.path>; // See endpoint: /frontend/init 

const socket = io(URL, {
	path: PATH,
	transports: ['websocket']
});
```


### Das Protokoll 
#### Nachrichtentypen (Events)
Alle derzeit implementierten Nachrichtentypen sind dem *enum socketMessageTypes* zu entnehmen.<br />

*Siehe: (https://github.com/rocketbeans/rbtv-apidoc/blob/master/socket/messageTypes.ts)*

Die Semantik des Nachrichtentyp-Namens definiert die Flussrichtung sowie den Namen der Nachricht selbst.

Beispiel:
```
 AC_HELLO
 || `- Typ der Nachricht: 'HELLO'
 | `-- Empf�nger der Nachricht: [C]lient
  ` -- Sender der Nachricht: [A]PI
```

Derzeit verwendete Ziele:
 - **A** => API
 - **C** => Client (Frontend, App, etc..)
 - **R** => Regietool
 
Der vollst�ndige Name der Nachricht entspricht dem Socket.IO Eventnamen (z.B: *AC_AUTHENTICATION_REQ*)

#### Payload (Nutzlast)

Sofern nicht anders spezifiziert, wird die Nutzlast einer Nachricht JSON kodiert �bertragen; 
es gibt f�r alle Nachrichtentypen ein entsprechend gleichnamiges Typescript Interface welches die Struktur des Payloads genauer definiert.

*Siehe: (https://github.com/rocketbeans/rbtv-apidoc/tree/master/socket/messages)*

#### Protokollversion

Nach erfolgreichem Verbindungsaufbau sendet die API die Nachricht "*AC_HELLO*".
Diese Nachricht beinhaltet die Protokollversion.

*Siehe: (https://github.com/rocketbeans/rbtv-apidoc/blob/master/socket/messages/misc/HelloPublic.ts)*

### Keepalive (erforderlich)

Die API sendet, um die Verbindung ordnungsgem�� aufrecht zu erhalten, regelm��ig (etwa alle 30 Sekunden) die Nachricht "*AC_PING*".
Eine ordnungsgem��e Client-Implementierung sollte umgehend mit "*AC_PONG*" antworten.

Beispiel:
```typescript
const socket = io( ... ); // Socket.IO - Client

socket.on('AC_PING', (<XX_PING>data) => { // Type XX_PING See: https://github.com/rocketbeans/rbtv-apidoc/blob/master/socket/messages/misc/HelloPublic.ts
	socket.emit('CA_PONG', <XX_PING>{ 
		id: data.id,
		tick: data.tick
	});
});
```

### Authentifizierung
Die Authentifizierung erfolgt mit einem g�ltigen Access Token 

*Siehe: (https://github.com/rocketbeans/rbtv-apidoc/blob/master/doc/OAuth%20%26%20Authorization.md)*

Eine ordnungsgem��e Client-Implementierung muss auf folgende Nachrichtentypen reagieren:

 - ***AC_AUTHENTICATION_REQ***<br />
   Die API signalisiert dem Client dass sie bereit zur Authentifizierung ist.<br />
   Per Definition kann dieses Event zu jedem Zeitpunkt kommen.<br />
   Authentifizierung erfolgt via *CA_AUTHENTICATION* (siehe unten)
   
 - ***AC_AUTHENTICATION_RENEW_TOKEN_REQ***<br />
   Die API signalisiert dem Client dass er seinen Access-Token zeitnah mit Hilfe des Refresh-Tokens erneuern sollte.<br />
   Dieses Event tritt etwa 120 Sekunden vor Ablauf des verwendeten Access-Tokens auf.<br />
   Nach erfolgter Erneuerung des Access-Tokens erfolgt eine normale Authentifizierung via *CA_AUTHENTICATION* (siehe unten)<br />
   Das Erneuern des Access-Tokens per Websocket ist nicht m�glich.


Authentifizierung via Access-Token ***CA_AUTHENTICATION***:<br />
Die Authentifizierung ist jederzeit m�glich, muss jedoch nach *AC_AUTHENTICATION_REQ* sowie zeitnah nach *AC_AUTHENTICATION_RENEW_TOKEN_REQ* erfolgen.<br />
Das Ergebnis der Authentifizierung wird dem Client mit der Nachricht *AC_AUTHENTICATION_RESULT* umgehend mitgeteilt.

*Siehe: (https://github.com/rocketbeans/rbtv-apidoc/blob/master/socket/messages/auth/AuthPublic.ts)*

Beispiel:
```typescript
const socket = io( ... ); // Socket.IO - Client

function handleAuthentication() {
	socket.emit('CA_AUTHENTICATION', <CA_AUTHENTICATION> {
		token: myFancyTokenStorage.getValidAccessToken(),
		appName: 'My Fancy Example App/2.1'	// App Name & Version Identifier
	});
}

var g_bInitialAuthorizationSucceeded = false;
function handleAuthenticationResult(<AC_AUTHENTICATION_RESULT> data){
	if(data.result !== true) {
		// - UI State Change
		// - Re-Authenticate the User via OAUTH 
		// ...
		alert('Socket: Authentifizierung fehlgeschlagen!');
		
	}else {
		// Authentication successful
		// depending on granted scopes some parts of the API
		// may or may not be available.
		// 
		
		if(!g_bInitialAuthorizationSucceeded){
			// - UI State Changes
			// - Request additional information from API utilizing the socket-API
			// etc ..
			
			
			g_bInitialAuthorizationSucceeded = true;
		}
					
	}
}

socket.on('AC_AUTHENTICATION_REQ', handleAuthentication);
socket.on('AC_AUTHENTICATION_RENEW_TOKEN_REQ', handleAuthentication);
socket.on('AC_AUTHENTICATION_RESULT', handleAuthenticationResult);
```

### Stream Info
Sowohl unauthentifiziert als auch authentifiziert werden dem Client �nderungen zum Livestream mitgeteilt.

 - ***AC_STREAM_YOUTUBECHANGED***<br />
   Die Livestream URL hat sich ge�ndert (Youtube Token)<br />

 - ***AC_STREAM_COUNT***<br />
   Die Zuschauerzahl hat sich ge�ndert<br />
   
 - ***AC_STREAM_INFO***<br />
   Die laufende Sendung hat sich ge�ndert<br />
   (beinhaltet au�erdem die aktuelle Livestream URL, sowie die aktuelle Zuschauerzahl)

Die oben genannten Events k�nnen jederzeit und in unbestimmter Reihenfolge auftreten.<br />
Eine ordnungsgem��e Client-Implementierung ber�cksichtigt sowohl *AC_STREAM_YOUTUBECHANGED* als auch *AC_STREAM_INFO* (insbesondere den *youtubeToken*).<br/>

Siehe: *(https://github.com/rocketbeans/rbtv-apidoc/blob/master/socket/messages/stream/StreamPublic.ts)*



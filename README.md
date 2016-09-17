# HackspaceWidget

This is an Android widget for Hackspace Catania.

The widget indicates the office state (open / close)

![alt tag](https://raw.github.com/davideagostini/Hackspace-widget/master/images/widget.jpg)


## Hackspace widget tutorial

Il **widget** è una semplice applicazione che tipicamente viene situata sulla home screen dei device Android. Scopo del nostro widget è quello di tenerci aggiornati sullo stato dell'Hackspace (Siamo in sede o Non siamo in sede).

I passi che di seguito illustreremo, sono validi per la creazione di un qualsiasi widget. E' bene specificare che nel nostro caso abbiamo fatto uso anche di un service, il motivo di questa scelta verrà spiegata successivamente.

Per lo sviluppo abbiamo utilizzato l'IDE di sviluppo **Android Studio** che è possibile scaricare al seguente [link.](https://developer.android.com/sdk/index.html)

Poi basta scaricare questo repository e importarlo all'interno di Android Studio.

![alt tag](https://raw.github.com/davideagostini/Hackspace-widget/master/images/image_1.png)

Adesso analizzeremo in dettaglio i passi necessari per lo sviluppo del nostro widget, analizzando in dettaglio i file creati.

## I cinque steps per la creazione del widget

1. creazione di un file XML denominato `widget_layout.xml` che rappresenta il layout del nostro widget sullo schermo
2. creazione un file XML denominato `widget_info.xml` che descrive le proprietà del widget come per esempio la dimensione e la frequenza di aggiornamento
3. creazione di un `AppWidgetProvider` che è utilizzato per costruire l'interfaccia utente del widget
4. creazione di un service denominato `UpdateStatusWidgetService.java` per effettuare una richiesta web ed aggiornare lo status del widget
5. infine è necessario registrare sia il widget che il service nell'`AndroidManifest.xml`

## 1. widget_layout.xml

All'interno del nostro file `app/src/main/res/layout/widget_layout.xml` è stato utilizzato un `RelativeLayout` [(qui maggiori dettagli)](http://developer.android.com/guide/topics/ui/layout/relative.html) per predisporre i nostri elementi in funzione degli altri elementi presenti nella  view. Gli elementi principali su cui porre l'attenzione sono:

1. un'`ImageView` che è un segnaposto del logo Hackspace

```xml
	<ImageView
	    android:id="@+id/icon"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:adjustViewBounds="true"
            android:contentDescription="@string/nd_desc"
            android:gravity="center_horizontal|center_vertical"
            android:scaleType="centerCrop"
            android:src="@drawable/ic_launcher" />
```
2. una `TextView`:

`statusText` che viene aggiornata in base alla response ottenuta. Solitamente lo status è **"Siamo in sede!"** oppure **"Non siamo in sede!"**. Inizialmente la nostra stringa è settata ad un valore di default che è il seguente *"loading..."*.
```xml
	<TextView
            android:id="@+id/statusText"
            style="@android:style/TextAppearance.Medium"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/status"
            android:textColor="#000000" />
```
Alla fine di questo processo, avremo il seguente risultato:

![alt tag](https://raw.github.com/davideagostini/Hackspace-widget/master/images/image_2.png)

## 2. widget_info.xml

Il file `app/src/main/res/xml/widget_info.xml` come già anticipato, setta le dimensioni del nostro widget. Ci sono delle regole specifiche per stabilire la dimensione del widget. Android arrangia la home screen in una griglia di dimensioni 4 x 4 mentre per i tablet offre una griglia di dimensioni 8 x 7. La documentazione specifica che per calcolare la dimensione del widget si deve seguire questa formula:

**dimensione = (numero di colonne o righe * 74) – 2**

Nel nostro caso vogliamo che il nostro widget abbia una dimensione di 4 x 1, da cui:

width = (4 * 74) – 2 = 294dp

height = (1 * 74) – 2 = 72dp

l'altro parametro `updatePeriodMillis` come è facile intuire si riferisce alla frequenza di aggiornamento del widget. Android di default effettua l'aggiornamento ogni 30 minuti per una questione di efficienza della batteria.
Nel caso in cui si voglia una frequenza di aggiornamento più bassa, è necessario ricorrere alla classe `AlarmManager` [(qui maggiori dettagli).](https://developer.android.com/reference/android/app/AlarmManager.html
)

```xml
    <?xml version="1.0" encoding="utf-8"?>
    <appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
        android:initialLayout="@layout/widget_layout"
        android:minHeight="72dp"
        android:minWidth="294dp"
        android:updatePeriodMillis="1800000" >

    </appwidget-provider>
```

## 3. AppWidgetProvider

`AppWidgetProvider` è una sottoclasse di `BroadcastReceiver` [(qui maggiori dettagli).](https://developer.android.com/reference/android/content/BroadcastReceiver.html
) Questa classe non fa altro che costruire l'intent per richiamare il nostro service `UpdateStatusWidgetService.java` ed aggiornare il widget tramite il service.

>**Nota:** Un **Intent** è un oggetto con cui una componente può richiedere l’esecuzione di un’azione da parte di un’altra componente.

>Un **Service** è una componente che viene eseguita in background, senza interazione diretta con l'utente. I services sono utilizzati per operazioni ripetitive e potenzialmente lunghe come: download da Internet, la ricerca di nuovi dati, l'elaborazione dei dati.

```java
    public class AppWidget extends AppWidgetProvider {

	    @Override
	    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
		    ComponentName thisWidget = new ComponentName(context, AppWidget.class);
		    int[] allWidgetIds = appWidgetManager.getAppWidgetIds(thisWidget);
		    // costruisce l'intent per chiamare il service
		    Intent intent = new Intent(context.getApplicationContext(), UpdateStatusWidgetService.class);
		    intent.putExtra(AppWidgetManager.EXTRA_APPWIDGET_IDS, allWidgetIds);
		    // aggiorna il widget tramite il service
		    context.startService(intent);
	    }
    }
```

## 4. UpdateStatusWidgetService.java

Infine ci rimane da analizzare in dettaglio il nostro service `UpdateStatusWidgetService.java` che viene eseguito in background e che aggiorna la nostra view.

>**Nota:** Si utilizza un service che gira in background perché i BroadcastReceivers sono soggetti ad errori del tipo “Application Not Responding” (ANR) che non fanno altro che chiedere all'utente di forzare la chiusura dell'applicazione nel caso in cui il task sta prendendo troppo tempo. Poichè una richiesta web a seconda delle circostanze potrebbe richiedere diversi secondi, l'utilizzo del service ci aiuta a superare questa problematica.

Il nostro service contiene il metodo `getHackspaceStatus(String url)` che semplicemente fa una richiesta ad un indirizzo web e processa la risposta.

```java
    private String getHackspaceStatus(String url) {

	}
```

Successivamente attraverso il componente `RemoteViews` abbiamo accesso alla `TextView` la quale viene aggiornata in base alla risposta ottenuta tramite il metodo `setTextViewText()`.

```java
    RemoteViews remoteViews = new RemoteViews(this.getApplicationContext().getPackageName(), R.layout.widget_layout);
    remoteViews.setTextViewText(R.id.statusText, status);
```

Aggiungiamo un *onClickListener* in maniera tale che quando clicchiamo sul widget presente sulla home screen forziamo l'aggiornamento. Questo viene fatto tramite `PendingIntent` e in particolare con il metodo `setOnClickPendingIntent()`. Una volta effettuato il nostro lavoro, aggiorniamo il nostro widget tramite il metodo `updateAppWidget()`.

```java
    PendingIntent pendingIntent = PendingIntent.getBroadcast(getApplicationContext(), 0, clickIntent, PendingIntent.FLAG_UPDATE_CURRENT);
    remoteViews.setOnClickPendingIntent(R.id.layout, pendingIntent);
		appWidgetManager.updateAppWidget(widgetId, remoteViews);
```

## 5. AndroidManifest.xml

Nel file `AndroidManifest.xml` registriamo il nostro service e il nostro receiver.
In particolare per il receiver specifichiamo che il nostro `AppWidgetProvider` accetta come azioni l'update (`ACTION_APPWIDGET_UPDATE`).

```xml
    <service android:name=".UpdateStatusWidgetService" />

    <receiver
        android:name="AppWidget"
        android:icon="@drawable/ic_launcher" >
        <intent-filter>
            <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
        </intent-filter>

       	<meta-data
            android:name="android.appwidget.provider"
            android:resource="@xml/widget_info" />

    </receiver>
```

Non dimentichiamo di inserire il permesso [(qui maggiori dettagli)](http://developer.android.com/reference/android/Manifest.permission.html
) di connessione alla rete internet per la nostra app.

`<uses-permission android:name="android.permission.INTERNET" />`

## Esportare la nostra APK per la distribuzione

Una delle importanti esigenze di distribuzione è che le applicazioni Android siano firmate digitalmente con un certificato che lo sviluppatore detiene. Questo meccanismo viene utilizzato per garantire l'autenticità dell'applicazione.

Per fare questo avremo bisogno di una chiave privata per firmare l'APK finale.

I passi sono i seguenti:

![alt tag](https://raw.github.com/davideagostini/Hackspace-widget/master/images/apk_1.png)

* La prima volta clicchiamo Create New e generiamo la nostra chiave, le volte successive faremo riferimento alla chiave già generata

![alt tag](https://raw.github.com/davideagostini/Hackspace-widget/master/images/apk_2.png)

![alt tag](https://raw.github.com/davideagostini/Hackspace-widget/master/images/apk_3.png)

![alt tag](https://raw.github.com/davideagostini/Hackspace-widget/master/images/apk_4.png)

alla fine di questo processo avremo la nostra apk pronta per essere distribuita.

## Conclusioni

Come è possibile notare, la creazione di un widget non è così difficile per chi ha delle conoscenze di programmazione.

L'obiettivo di questo tutorial era quello di dare le basi a chiunque voglia cimentarsi in questo campo, per questo ho introdotto ed utilizzato solo i concetti essenziali.

Qualsiasi domanda o feedback sono ben accetti.

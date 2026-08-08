# TOGA - Camera

### Cél:

Olyan Android app létrehozása, amivel:

* Fotókat lehet készíteni vele anélkül, hogy az alkalmazást "elhagynánk"
* A készített fényképek nem az Android Fényképgalériába kerülnek, hanem egy erre a célra kialakított adatbázisba. (további lehetőségek: a kép EXIF adatainak kiolvasása, esetleg tárolása)

### Lehetséges felhasználási területek:

* Minden olyan alkalmazási terület, ahol az adatgyűjtés szerves része a fotódokumentumok (valamely helyszíni esemény/tárgy képeinek) tárolása

### A megvalósítás eszközei:

Python eszközök: Beeware-Toga, Briefcase, Flask, SQLite
Valamilyen Javascript könyvtár (jelen esetben: Webix)

### Implementáció

1. Toga (0.5.6) Python kód

* A WebView-t a háttérben futó Flask szerver irányítja (url="http://127.0.0.1:5000"), a toga triggereli a `trigger_camera_action` eseményt.

```python
"""
Fotózás és fényképtárolás - MacOs/Android app plan
"""
import asyncio
import base64
import toga
import threading
from toga.style import Pack
from flask import Flask, render_template, jsonify, request

class PhotoDB(toga.App):
    '''
     -  indítás:
    '''
    def startup(self):
        # 1. Flask inicializálása
        self.flask_app = Flask(__name__)
        self.setup_flask_routes()
      
        # Flask indítása háttérszálon
        threading.Thread(
            target=lambda: self.flask_app.run(host='127.0.0.1', port=5000, debug=False, use_reloader=False), 
            daemon=True
        ).start()

        # 2. WebView a Flask szerverre irányítva, URL figyeléssel
        self.webview = toga.WebView(
            style=Pack(flex=1),
            url="http://127.0.0.1:5000"
        )

        self.main_window = toga.MainWindow(title=self.formal_name)
        self.main_window.content = self.webview
        self.main_window.show()
    '''
     -  Flask route-ok:
    '''
    def setup_flask_routes(self):
        @self.flask_app.route('/')
        def index():
            # Ez rendereli ki a Webix-es template-et
            return render_template('index.html')

        @self.flask_app.route('/api/trigger-camera', methods=['POST'])
        def route_trigger_camera():
            # ÚJ MEGOLDÁS TOGA 0.5.6-HOZ:
            # A Flask száláról biztonságosan átküldjük a feladatot a Toga aszinkron Event Loop-jába
            def launch_task():
                asyncio.create_task(self.trigger_camera_action())

            # A call_soon_threadsafe garantálja, hogy a Toga főszála azonnal és biztonságosan 
            # végrehajtja a launch_task() függvényt, ami elindítja az aszinkron kamerát
            self.loop.call_soon_threadsafe(launch_task)

            return jsonify({"status": "initiated"})

        @self.flask_app.route('/api/save-db', methods=['POST'])
        def save_to_db():
            data = request.json
            image_base64 = data.get('image')
          
            # Ide jön a DB mentési logika (SQLite, PostgreSQL stb.)
            # ...
            return jsonify({"status": "stored_in_db", "message": "Sikeres mentés az adatbázisba!"})

    async def trigger_camera_action(self, app=None, **kwargs):
        try:
            if not self.camera.has_permission:
                await self.camera.request_permission()

            # Ez a sor nyitja meg a Mac FaceTime kameráját, vagy az Android kameráját
            photo = await self.camera.take_photo()
          
            if photo:
                # Kép konvertálása Base64 stringgé (Toga 0.5.6)
                raw_bytes = photo.data
                base64_str = base64.b64encode(raw_bytes).decode('utf-8')
              
                # Visszalőjük a Webix globális függvényének a kész képet
                js_code = f"onNativePhotoSuccess('{base64_str}');"
                self.webview.evaluate_javascript(js_code)
            else:
                self.webview.evaluate_javascript("onNativePhotoError('No shot');")
        except Exception as e:
            self.webview.evaluate_javascript(f"onNativePhotoError('{str(e)}');")


def main():
    return PhotoDB()
```

2. HTML / Webix JS oldal (templates/index.html)

Némi magyarázat:

Minthogy a Toga Python kód hamarabb próbálja meg lefutattatni az `onNativePhotoSuccess` JavaScript függvényt a WebView-ban, így a JavaScript függvényeket explicit módon rá kell kötnünk a globális window objektumra (lásd `config()` rész)

```HTML
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title></title>
	<!-- This is important to get the correct canvas size on mobile devices -->
	<meta name='viewport' content='width=device-width, user-scalable=no, minimum-scale=1.0, maximum-scale=1.0'/>
	<!-- Webix Library -->
	<script type="text/javascript" src="{{ url_for('static', filename='assets/webix.js') }}"></script>
	<script type="text/javascript" src="{{ url_for('static', filename='assets/webix-jet.js') }}"></script>
	<link rel="stylesheet" type="text/css" href="{{ url_for('static', filename='assets/webix.css') }}">
	<!-- Material Design Icons -->
	<link rel="stylesheet" type="text/css" href="{{ url_for('static', filename='assets/skins/materialdesignicons.css') }}">
</head>
<body>
	<script type="text/javascript">
	var JetApp = webix.jet.JetApp
	var JetView = webix.jet.JetView 

	class TopView extends JetView {
		config() {

			window.onNativePhotoSuccess = function(base64Data) {
				$$("shot_btn").hideProgress();
			
				// Megjelenítjük a Webix template-ben a képet
				$$("preview").setHTML(`<img src="data:image/jpeg;base64,${base64Data}" style="max-height:100%; max-width:100%;">`);
			
				// Webix AJAX hívás a Flask felé a DB mentéshez
				webix.ajax().headers({
					"Content-Type": "application/json"
				}).post("/api/save-db", JSON.stringify({ image: base64Data }))
				.then(function(response){
					let res = response.json();
					webix.message(res.message);
				})
				.catch(function(err){
					webix.message({ type:"error", text:"DB mentési hiba: " + err.statusText });
				});
			};

			window.onNativePhotoError = function(errorMsg) {
				$$("shot_btn").hideProgress();
				webix.message({ type:"error", text: "Kamera hiba: " + errorMsg });
			};

			return {
				type:"wide",
				padding:10,
				rows:[
					{ view:"template", type:"header", template:"Android kamera teszt" },
					{ 
						view:"button", 
						id:"shot_btn", 
						value:"📸 Lőjj egy képet", 
						css:"webix_primary",
						click: function() {
							webix.extend($$("shot_btn"), webix.ProgressBar);
							$$("shot_btn").showProgress();
						
							// AJAX-szal indítjuk a kamerát a Flask-en keresztül
							webix.ajax().post("/api/trigger-camera");
						}
					},
					{ view:"template", id:"preview", template:"Nincs fénykép", autoheight:true }
				]
			}
		}
	}
	// Applikáció indítása
	webix.ready(() => {
		webix.ui.fullScreen()

		if (!webix.env.touch && webix.env.scrollSize && webix.CustomScroll)
			webix.CustomScroll.init()

		const app = new JetApp({
			start: "/top",
			debug: true,
			views: {
				top: TopView
			}
		})
		app.render()
	})
    </script>
</body>
</html>
```

Miért kiváló ez a hibrid felépítés?

* Kihasználja a Flask architektúrát: A Javascript rész AJAX-szal kommunikál a Flask-kel, így bármely DB-kezelő (SQLite, PostgreSQL stb.) alkalmazható adattárolásra.
* A kamera megnyitását a Toga natív, stabil API-jára bízza
* Zökkenőmentes adatfolyam: a fénykép átadása (Toga -> WebView JS -> Flask Route -> DB) teljesen a memóriában és lokális HTTP hívásokon keresztül zajlik, így nincs szükség bonyolult Androidos fájl-jogosultságok konfigurálására.

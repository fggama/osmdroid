# Implementación OSMDROID
para aplicaciones Kotlin en Android

## AndroidManifest.xml
```
<provider
	android:name="androidx.core.content.FileProvider"
	android:authorities="com.epsas.genesys_medidores.fileprovider"
	android:exported="false"
	android:grantUriPermissions="true">
	<meta-data
		android:name="android.support.FILE_PROVIDER_PATHS"
		android:resource="@xml/file_paths" />
</provider>

```

### _res/xml/file_paths.xml_
```
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <files-path name="fotografias" path="images/"/>
    <files-path name="archivos" path="archivos/"/>
    <files-path name="osmdroid" path="osmdroid/"/>
    <root-path
        name="root"
        path="."/>
</paths>
```

## Archivos
__Mapas__:  
``/data/data/com.epsas.genesys_medidores/files/osmdroid/osmdroid.zip``

__Tiles__ (cache):  
``/data/data/com.epsas.genesys_medidores/files/osmdroid/tiles``

## ViewModel
```
class ConfigActivityViewModel(context: Context) : ViewModel() {
    var osmFile: File

    private var fileName: String = "osmdroid.zip"
    var tamanio: MutableLiveData<Float> = MutableLiveData(0f)
    var isFileReadyObserver = MutableLiveData<Boolean>()

    init {
        val imagePath = File(context.filesDir, "osmdroid")
        osmFile = File(imagePath, fileName)
    }

    fun downloadOSMFile(osmUrl: String) {
        thread {
            val retroServiceInterface =
                RetrofitClient.getClientOSM().create(DownloadService::class.java)

            retroServiceInterface.downloadOSM(osmUrl).enqueue(object : Callback<ResponseBody> {
                override fun onFailure(call: Call<ResponseBody>, t: Throwable) {
                    isFileReadyObserver.postValue(false)
                    Logger.e("downloadOSMFile", t.message.toString())
                }

                override fun onResponse(
                    call: Call<ResponseBody>,
                    response: Response<ResponseBody>
                ) {
                    if (response.isSuccessful) {
                        val result = response.body()?.byteStream()
                        result?.let {
                            writeToFile(it)
                        } ?: kotlin.run {
                            isFileReadyObserver.postValue(false)
                        }
                    } else
                        isFileReadyObserver.postValue(false)
                }
            })
        }
    }

    private fun writeToFile(inputStream: InputStream) {
        try {
            val fileReader = ByteArray(4096)
            var fileSizeDownloaded = 0
            val fos: OutputStream = FileOutputStream(osmFile.absolutePath)
            do {
                val read = inputStream.read(fileReader)
                if (read != -1) {
                    fos.write(fileReader, 0, read)
                    fileSizeDownloaded += read
                }
            } while (read != -1)
            fos.flush()
            fos.close()
            isFileReadyObserver.postValue(true)
            Logger.i("writeToFile", "Destino: ${osmFile.absolutePath}")
        } catch (e: IOException) {
            Logger.e("writeToFile", "====IOException : $e")
            isFileReadyObserver.postValue(false)
        }
    }

    /**
     *  Si existe el archivo ->
     *  /data/data/com.epsas.genesys_medidores/files/osmdroid/osmdroid.zip
     */
    fun tieneArchivoMapas(): Boolean {
        try {
            if (osmFile.isFile) {
                tamanio.value = osmFile.length() / 1024f / 1024f
                return true
            }
        } catch (ex: Exception) {
            Logger.e("tieneArchivoMapas", ex.message.toString())
        }
        return false
    }
}
```

## Main Activity
```
WindowCompat.setDecorFitsSystemWindows(window, true)

```


## Map Activity

```
private val onlineTileSourceBase: OnlineTileSourceBase = XYTileSource(
	"HttpMapnik",
	0, 25, 256, ".png", arrayOf(
		"http://a.tile.openstreetmap.org/",
		"http://b.tile.openstreetmap.org/",
		"http://c.tile.openstreetmap.org/"
	),
	"© OpenStreetMap contributors"
)
private lateinit var viewModelOSM: ConfigActivityViewModel
```
### onCreate
```
Configuration.getInstance().userAgentValue = applicationContext.packageName

val ctx = application
Configuration.getInstance().load(ctx, PreferenceManager.getDefaultSharedPreferences(ctx))

viewModelOSM =
	ViewModelProvider(this@MapActivity, object : ViewModelProvider.NewInstanceFactory() {
		override fun <T : ViewModel> create(modelClass: Class<T>): T {
			@Suppress("UNCHECKED_CAST")
			return ConfigActivityViewModel(ctx) as T
		}
	})[ConfigActivityViewModel::class.java]

boundingBox = BoundingBox.fromGeoPoints(emptyList<GeoPoint>())

val osmConf = Configuration.getInstance()
osmConf.osmdroidBasePath = File(viewModelOSM.osmFile.absolutePath)
val tileCache = File(filesDir.absolutePath, "osmdroid/tiles")
if (!tileCache.exists()) {
	tileCache.mkdirs()
}
osmConf.osmdroidTileCache = tileCache
tileSource = TileSourceFactory.MAPNIK

try {
	tileProvider = MapTileProviderBasic(ctx)
} catch (_: Exception) {

}
tileProvider.tileSource = tileSource
setContentView(R.layout.activity_map)

/**
 * MapView settings
 */
binding.mapview.setTileSource(tileSource)
binding.mapview.zoomController.setVisibility(CustomZoomButtonsController.Visibility.NEVER)
binding.mapview.isClickable = true
binding.mapview.isTilesScaledToDpi = true
binding.mapview.setMultiTouchControls(true)
binding.mapview.setUseDataConnection(false)
binding.mapview.keepScreenOn = true
binding.mapview.maxZoomLevel = 25.0
binding.mapview.minZoomLevel = 5.0

mapController = binding.mapview.controller
mapController.setZoom(14.0)

tilesOverlay = TilesOverlay(tileProvider, ctx)
binding.mapview.overlays.add(tilesOverlay)

```

## Global Class
```
const val servidorOSM: String = "http://190.181.5.90:5055/"
const val osmUrl: String = "http://190.181.5.90:5055/mapas"

fun archivoOSM(context: Context): Boolean{
	if (File(context.filesDir, "osmdroid").isDirectory) {
		return File(context.filesDir, "osmdroid/osmdroid.zip").isFile
	}
	return false
}
	
```

## Download Activity (osmdroid.zip)
```
private lateinit var viewModel: ConfigActivityViewModel

private fun initDownload() {
	viewModel =
		ViewModelProvider(this@ConfigActivity, object : ViewModelProvider.NewInstanceFactory() {
			override fun <T : ViewModel> create(modelClass: Class<T>): T {
				return ConfigActivityViewModel(fileDir = filesDir) as T
			}
		})[ConfigActivityViewModel::class.java]

	viewModel.isFileReadyObserver.observe(this@ConfigActivity) {
		setDialog(false)
		if (!it) {
			val builder = AlertDialog.Builder(this@ConfigActivity)
			builder.setTitle("Descarga osmdroid.zip")
			builder.setMessage("No se pudo descargar el archivo")
			builder.setNeutralButton("Aceptar", null)
			builder.setIcon(www.sanju.motiontoast.R.drawable.ic_error_)
			val dialog = builder.create()
			dialog.show()
		} else {
			val builder = AlertDialog.Builder(this@ConfigActivity)
			builder.setTitle("Descarga osmdroid.zip")
			builder.setMessage("Archivo descargado!")
			builder.setNeutralButton("Aceptar", null)
			builder.setIcon(www.sanju.motiontoast.R.drawable.ic_check_green)
			var dialog = builder.create()
			dialog.show()

			try {
				if (Global.archivoOSM(this)) {
					val archivo = "osmdroid.zip (${
						File(
							filesDir,
							"osmdroid/osmdroid.zip"
						).length() / 1024 / 1024
					} MB)"
					binding.archivoMapas.text = archivo
					binding.archivoMapas.setCompoundDrawablesWithIntrinsicBounds(
						0,
						0,
						0,
						0
					)
				}
			} catch (e: Exception) {
				builder.setMessage(e.message.toString())
				builder.setIcon(www.sanju.motiontoast.R.drawable.ic_error_)
				dialog = builder.create()
				dialog.show()
			}
		}
	}

	/**
	 * Descarga
	 */
	setDialog(true, "Descargando...")
	viewModel.downloadOSMFile(Global.osmUrl)
}
```

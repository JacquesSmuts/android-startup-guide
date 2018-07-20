# android-startup-guide
This is to keep a list of all those little things one needs to do or import or consider when starting a new Android project.

## Gradle
You probably want to import these on top of the defaults:

    // dependency injection
    api 'com.github.salomonbrys.kodein:kodein:4.1.0'

    //Jake Wharton
    implementation 'com.jakewharton.timber:timber:4.7.1'
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.5.4'
    releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.5.4'

    // convenience/utils
    implementation 'com.github.matecode:Snacky:1.0.3'
    implementation 'com.google.code.gson:gson:2.8.5'

    // rx
    implementation 'io.reactivex.rxjava2:rxandroid:2.0.1'
    implementation 'io.reactivex.rxjava2:rxjava:2.1.13'

and add this:

    productFlavors {
        dev {
            minSdkVersion 23
            applicationIdSuffix = ".dev"
            resValue "string", "app_name", (rootProject.ext.appName + "_dev")
            resValue "string", "leak_canary_display_activity_label", (rootProject.ext.appName  + "_dev_leaks")
        }
        beta {
            //applicationIdSuffix = ".beta"
            //resValue "string", "app_name", (rootProject.ext.appName + "_beta")
            resValue "string", "leak_canary_display_activity_label", (rootProject.ext.appName  + "_beta_leaks")
        }
        prod {
            resValue "string", "app_name", rootProject.ext.appName
            resValue "string", "leak_canary_display_activity_label", (rootProject.ext.appName + "_leaks")
        }
    }    
    
    ext {
        support_version = "27.1.1"
        firebase_version = "16.0.1" 
    }
    
    
## Application
And create your own Application class which looks something this

    class MyApp: Application(){

        companion object {
            lateinit var kodein: Kodein
                private set
        }

        override fun onCreate() {
            super.onCreate()
            if (LeakCanary.isInAnalyzerProcess(this)) {
                // This process is dedicated to LeakCanary for heap analysis.
                // You should not init your app in this process.
                return;
            }
            if (BuildConfig.FLAVOR == "dev") {
                LeakCanary.install(this)
            }
            if (BuildConfig.DEBUG) {
                Timber.plant(DebugTree())
            }

            val that = this
            kodein = Kodein {
                bind<Application>() with instance(that)
                bind<Context>() with instance(applicationContext)
                bind<AuthService>() with singleton { AuthService() }
                bind<ApiService>() with singleton { ApiService(instance()) }
            }
        }
    }

## BaseActivity
With a BaseActivity which looks a bit like this:

    abstract class BaseActivity : AppCompatActivity() {

        protected val apiService: ApiService by KinApp.kodein.lazy.instance()

        protected val rxSubs : io.reactivex.disposables.CompositeDisposable by lazy {
            io.reactivex.disposables.CompositeDisposable()
        }

        override fun onResume() {
            super.onResume()

            rxSubs.add(RxBus.listen(ProgressEvent::class.java).subscribe(){
                showProgress(it.shouldShowProgressBar)
            })

        }

        /**
         * All activities require some sort of loading state
         */
        abstract fun showProgress(shouldShowProgress: Boolean = true)

        protected fun hideProgress(){
            showProgress(false)
        }

        override fun onPause() {
            rxSubs.clear()
            super.onPause()
        }
    }    
    

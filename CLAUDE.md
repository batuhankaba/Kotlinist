# CLAUDE.md — Proje Kuralları

Bu dosya projenin mimari anayasasıdır. Kod üretirken, düzenlerken veya öneri sunarken buradaki
kurallara **istisnasız** uy. Bir kuralın dışına çıkman gerekiyorsa önce gerekçesini belirt ve onay iste.

---

## 1. Proje Kimliği

Rick & Morty API'sini tüketen, **liste** ve **detay** olmak üzere iki ekranlı bir Android uygulaması.
Amaç: junior seviye portfolyo projesi. Kod okunabilirliği ve mimari tutarlılık, hız ve kısa yoldan
daha önemlidir.

**Dil:** Kotlin · **UI:** XML + ViewBinding · **Min SDK:** 24

---

## 2. Teknoloji Yığını

| Alan | Seçim |
|---|---|
| Mimari | MVVM + Clean Architecture (tek modül, paket bazlı katmanlar) |
| UI | XML layout + **ViewBinding** |
| Ekran yapısı | Single Activity + Multi Fragment |
| Navigasyon | Navigation Component + SafeArgs |
| DI | Hilt |
| Ağ | Retrofit + OkHttp + Gson |
| Asenkron | Coroutines + Flow |
| Liste | ListAdapter + DiffUtil |
| Görsel | Glide |
| Build | Gradle **Kotlin DSL** (`.kts`) + Version Catalog (`libs.versions.toml`) |
| Yönelim | **Sadece portrait** |

---

## 3. Paket Yapısı

```
com.batuhankaba.kotlinist
├── core/
│   ├── base/          UseCase, FlowUseCase, BaseViewModel, BaseFragment
│   └── ext/           ortak extension'lar (collectState, collectEvent, mapper'lar)
├── data/
│   ├── remote/api/    ApiService
│   ├── remote/model/  ...Response modelleri
│   └── repository/    ...RepositoryImpl
├── domain/
│   ├── model/         ...UI modelleri
│   ├── repository/    ...Repository (interface)
│   └── usecase/       ...UseCase / ...FlowUseCase
├── presentation/
│   ├── list/          Fragment + ViewModel + Adapter
│   └── detail/
└── di/                NetworkModule, RepositoryModule
```

**Bağımlılık yönü:** `presentation → domain ← data`
Presentation katmanı `data` paketini **asla** import etmez.

> Bilinen taviz: mapping UseCase içinde yapıldığı için `domain` katmanı `data/remote/model`
> paketindeki Response sınıflarını import eder. Bu bilinçli bir karardır, sorgulanmasına gerek yok.

---

## 4. İsimlendirme Kuralları

| Tip | Kural | Örnek |
|---|---|---|
| Servis cevabı | `...Response` | `CharacterListResponse`, `CharacterDetailResponse` |
| UI modeli | `...UI` | `CharacterListUI`, `CharacterDetailUI` |
| UseCase | Fiil ile başlar | `GetCharacterListUseCase` |
| Repository | `...Repository` / `...RepositoryImpl` | `CharacterRepository` |
| Fragment | `...Fragment` | `CharacterListFragment` |
| ViewModel | `...ViewModel` | `CharacterListViewModel` |
| Adapter | `...Adapter` | `CharacterListAdapter` |
| Layout | `fragment_...`, `item_...` | `fragment_character_list.xml` |

**DTO terimi kullanılmaz.** Servis cevapları her zaman `Response` son ekiyle adlandırılır.

---

## 5. Base Sınıflar

Bu imzalar sabittir. Değiştirme, genişletme gerekiyorsa önce sor.

### 5.1 UseCase (suspend, Flow dönmez)

```kotlin
abstract class UseCase<in P, out R> {
    abstract suspend operator fun invoke(params: P): R
}
```

### 5.2 FlowUseCase (ağ istekleri, Flow döner)

```kotlin
abstract class FlowUseCase<in P, out R> {
    abstract operator fun invoke(params: P): Flow<R>
}
```

### 5.3 BaseViewModel

```kotlin
abstract class BaseViewModel : ViewModel() {

    private val _loadingState = MutableStateFlow(false)
    val loadingState: StateFlow<Boolean> = _loadingState.asStateFlow()

    private val _errorState = MutableSharedFlow<String>()
    val errorState: SharedFlow<String> = _errorState.asSharedFlow()

    protected fun showLoading() { _loadingState.value = true }

    protected fun hideLoading() { _loadingState.value = false }

    protected suspend fun defaultErrorHandler(throwable: Throwable) {
        _errorState.emit(throwable.message.orEmpty())
    }

    protected fun serviceCall(block: suspend CoroutineScope.() -> Unit): Job =
        viewModelScope.launch(block = block)
}
```

`serviceCall` **sadece scope açar.** İçine try/catch, loading yönetimi veya dispatcher gömülmez —
bunlar Flow zincirinde elle yönetilir.

### 5.4 BaseFragment

```kotlin
abstract class BaseFragment<VB : ViewBinding, VM : BaseViewModel>(
    private val inflateBinding: (LayoutInflater, ViewGroup?, Boolean) -> VB
) : Fragment() {

    private var _binding: VB? = null
    protected val binding: VB get() = _binding!!

    protected abstract val viewModel: VM

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?
    ): View {
        _binding = inflateBinding(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        initViews()
        initListeners()
        initObservers()
        observeBaseStates()
    }

    private fun observeBaseStates() {
        collectState(viewModel.loadingState) { isLoading -> onLoadingChanged(isLoading) }
        collectEvent(viewModel.errorState) { message -> showError(message) }
    }

    protected open fun initViews() {}
    protected abstract fun initListeners()
    protected abstract fun initObservers()

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

Binding **reflection ile inflate edilmez**, constructor'a metot referansı geçilir.

### 5.5 Ortak Extension'lar (`core/ext/FlowExt.kt`)

```kotlin
fun <T> Fragment.collectState(flow: StateFlow<T>, action: (T) -> Unit) {
    viewLifecycleOwner.lifecycleScope.launch {
        viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            flow.collect { action(it) }
        }
    }
}

fun <T> Fragment.collectEvent(flow: SharedFlow<T>, action: (T) -> Unit) {
    viewLifecycleOwner.lifecycleScope.launch {
        viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            flow.collect { action(it) }
        }
    }
}
```

Fragment içinde **çıplak `lifecycleScope.launch { flow.collect { } }` yazılmaz.** Her zaman bu iki
extension kullanılır.

---

## 6. UseCase Kuralları

- Parametreler **UseCase'in kendi içinde** `data class Params` olarak tanımlanır, ayrı dosyaya çıkarılmaz.
- Parametresiz UseCase'lerde de `Params` tanımlanır (`data class Params(val page: Int = 1)` gibi).
- Response → UI mapping **UseCase içinde** yapılır. Repository mapping yapmaz.
- Ağ isteği içeren her UseCase `FlowUseCase`'den, geri kalanlar `UseCase`'den kalıtır.

```kotlin
class GetCharacterListUseCase @Inject constructor(
    private val repository: CharacterRepository
) : FlowUseCase<GetCharacterListUseCase.Params, List<CharacterListUI>>() {

    data class Params(val page: Int = 1)

    override fun invoke(params: Params): Flow<List<CharacterListUI>> =
        repository.getCharacters(params.page).map { response ->
            response.results.orEmpty().map { it.toCharacterListUI() }
        }
}
```

---

## 7. Data Katmanı Kuralları

- **Response modellerinin tüm alanları nullable'dır.** Gson reflection kullandığı için non-null
  tanımlanan bir alan JSON'da yoksa sessizce `null` atanır ve null güvenliği bozulur.
  Non-null'a dönüşüm mapping sırasında yapılır (`?: ""`, `.orEmpty()`, `?: 0`).
- Alan adları `@SerializedName` ile eşlenir.
- Repository **interface**'i `domain/repository`, **implementasyonu** `data/repository` altındadır.
- Repository ham `Response` döner, mapping yapmaz.

```kotlin
data class CharacterListResponse(
    @SerializedName("info") val info: InfoResponse? = null,
    @SerializedName("results") val results: List<CharacterResponse>? = null
)
```

---

## 8. Hilt Kuralları

- Application sınıfı `@HiltAndroidApp`, Fragment'lar `@AndroidEntryPoint`, ViewModel'ler `@HiltViewModel`.
- 3. parti sınıflar (`OkHttpClient`, `Retrofit`, `ApiService`) → `NetworkModule` içinde `@Provides`.
- Interface → implementasyon bağları → `RepositoryModule` içinde `@Binds` (abstract class).
- Uygulama ömrü boyunca tek olması gerekenler `@Singleton`.
- Aynı tipten birden fazla bağımlılık gerekirse `@Qualifier` tanımlanır, `@Named` string'i tercih edilmez.

---

## 9. ViewModel Kuralları

- Her ViewModel `BaseViewModel`'den kalıtır.
- State `private val _x = MutableStateFlow(...)` + `val x: StateFlow<...> = _x.asStateFlow()` kalıbıyla açılır.
  Dışarıya **asla** mutable tip sızmaz.
- Ağ çağrıları `serviceCall { }` içinde yapılır.
- Loading ve hata yönetimi **Flow zincirinde elle** yazılır:

```kotlin
@HiltViewModel
class CharacterListViewModel @Inject constructor(
    private val getCharacterListUseCase: GetCharacterListUseCase
) : BaseViewModel() {

    private val _characters = MutableStateFlow<List<CharacterListUI>>(emptyList())
    val characters: StateFlow<List<CharacterListUI>> = _characters.asStateFlow()

    fun getCharacters(page: Int = 1) = serviceCall {
        getCharacterListUseCase(GetCharacterListUseCase.Params(page))
            .flowOn(Dispatchers.IO)
            .onStart { showLoading() }
            .onCompletion { hideLoading() }
            .catch { defaultErrorHandler(it) }
            .collect { _characters.value = it }
    }
}
```

**Zincir sırası sabittir:** `flowOn` → `onStart` → `onCompletion` → `catch` → `collect`.
`flowOn` sadece kendinden yukarısını etkiler; UseCase ve repository IO'da, state güncellemesi Main'de çalışır.

Navigasyon argümanı gerekiyorsa `SavedStateHandle` üzerinden okunur:

```kotlin
private val characterId: Int = checkNotNull(savedStateHandle["characterId"])
```

---

## 10. Fragment Kuralları

- Her Fragment `BaseFragment<VB, VM>`'den kalıtır ve `@AndroidEntryPoint` ile işaretlenir.
- ViewModel `override val viewModel: XViewModel by viewModels()` ile alınır.
- **İlk veri isteği `onCreate` içinde tetiklenir**, `onViewCreated` içinde değil.
  Gerekçe: Navigation Component `replace()` kullandığı için geri dönüşte `onViewCreated` yeniden
  çalışır ve istek tekrarlanır. `onCreate` Fragment instance'ı yaşadığı sürece bir kez çalışır.
- Fragment argümanı `by navArgs()` ile okunur.
- `initObservers()` sadece flow dinleme, `initListeners()` sadece tıklama/kullanıcı etkileşimi içerir.

```kotlin
@AndroidEntryPoint
class CharacterListFragment : BaseFragment<FragmentCharacterListBinding, CharacterListViewModel>(
    FragmentCharacterListBinding::inflate
) {
    override val viewModel: CharacterListViewModel by viewModels()

    private val adapter = CharacterListAdapter { characterId -> navigateToDetail(characterId) }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        viewModel.getCharacters()
    }

    override fun initViews() {
        binding.recyclerView.adapter = adapter
    }

    override fun initListeners() { /* ... */ }

    override fun initObservers() {
        collectState(viewModel.characters) { adapter.submitList(it) }
    }
}
```

---

## 11. RecyclerView Kuralları

- `ListAdapter` + `DiffUtil.ItemCallback` kullanılır. **`notifyDataSetChanged()` yasak.**
- Veri `submitList()` ile güncellenir, adapter'a constructor'dan liste geçilmez.
- ViewHolder içinde `findViewById` yok, ViewBinding kullanılır.
- Tıklama olayı adapter'a **lambda** olarak geçilir; adapter Fragment'ı veya ViewModel'i tanımaz.
- Scroll pozisyonunun korunması için `stateRestorationPolicy` ayarlanır:

```kotlin
class CharacterListAdapter(
    private val onItemClick: (Int) -> Unit
) : ListAdapter<CharacterListUI, CharacterListAdapter.ViewHolder>(DIFF_CALLBACK) {

    init {
        stateRestorationPolicy = StateRestorationPolicy.PREVENT_WHEN_EMPTY
    }

    companion object {
        private val DIFF_CALLBACK = object : DiffUtil.ItemCallback<CharacterListUI>() {
            override fun areItemsTheSame(old: CharacterListUI, new: CharacterListUI) = old.id == new.id
            override fun areContentsTheSame(old: CharacterListUI, new: CharacterListUI) = old == new
        }
    }
}
```

---

## 12. Navigation Kuralları

- Tek `nav_graph.xml`, tek `NavHostFragment`, tek Activity.
- Ekranlar arası geçiş **action** ile yapılır, doğrudan `FragmentTransaction` kullanılmaz.
- Argümanlar **SafeArgs** ile taşınır. `Bundle`'a elle key yazılmaz.
- **Sadece ID taşınır**, tüm nesne Parcelable olarak taşınmaz.
- Toolbar başlıkları `nav_graph` içindeki `label` alanından otomatik gelir
  (`setupActionBarWithNavController`). Ayrı toolbar bileşeni yazılmaz.

---

## 13. Kaynak ve Stil Kuralları

- **Hardcoded string yasak.** Tüm metinler `strings.xml`'de.
- Tekrar eden renk, boyut ve stiller `colors.xml`, `dimens.xml`, `styles.xml`'de tanımlanır.
- Layout'larda `ConstraintLayout` tercih edilir, iç içe `LinearLayout` yığınından kaçınılır.
- Glide kullanımında `placeholder` ve `error` görseli her zaman tanımlanır.

---

## 14. Build ve Release Kuralları

- Bağımlılıklar `libs.versions.toml` üzerinden yönetilir, `build.gradle.kts` içine sürüm yazılmaz.
- `BASE_URL` `buildConfigField` ile Gradle'dan verilir, koda gömülmez.
- `HttpLoggingInterceptor` **sadece debug** build'de eklenir (`if (BuildConfig.DEBUG)`).
- `INTERNET` izni manifest'te tanımlıdır.
- ProGuard sade tutulur: Gson reflection kullandığı için Response modelleri `@Keep` ile korunur.
  Gereksiz kural eklenmez.

---

## 15. Kesin Yasaklar

- `findViewById` — ViewBinding kullan
- DataBinding — sadece ViewBinding
- Jetpack Compose — bu proje tamamen XML
- `notifyDataSetChanged()` — `submitList` kullan
- Fragment'ta çıplak `lifecycleScope.launch { collect }` — `collectState` / `collectEvent` kullan
- `onDestroyView`'da binding null'lamayı atlamak
- Presentation katmanında `...Response` tipi kullanmak
- Response modellerinde non-null alan tanımlamak
- `!!` operatörü (BaseFragment'taki `binding` getter'ı hariç)
- ViewModel'de `Context` veya `View` referansı tutmak
- Dışarıya `MutableStateFlow` / `MutableSharedFlow` açmak
- Ağ isteğini `onViewCreated` içinde tetiklemek

---

## 16. Şimdilik Kapsam Dışı

Aşağıdakiler bilinçli olarak ertelendi. İstenmediği sürece ekleme, ekleme önerisi de sunma:

unit test, UI test, ktlint/detekt, `Resource`/`sealed` sonuç sarmalayıcı, dispatcher injection,
Room / offline cache, Paging 3, multi-module yapı, Compose.

---

## 17. Kod Üretirken Beklenen Davranış

- Yeni bir ekran istendiğinde **tam set** üret: Response → UI model → mapper → UseCase → Repository
  metodu → ApiService metodu → ViewModel → Fragment → layout → nav_graph kaydı.
- Var olan bir kalıp varsa onu taklit et, yeni bir kalıp icat etme.
- Bir şey belirsizse tahmin etme, sor.
- Açıklama isteniyorsa kavramı **neden öyle yaptığımızı** da anlat; bu proje aynı zamanda bir
  öğrenme projesidir.

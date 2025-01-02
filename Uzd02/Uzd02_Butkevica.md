Otrais uzdevums: vektordati, to ģeometrijas, atribūti un failu formāti
================
Jekaterīna Butkeviča,
2024. gada 31. decembris

# 1. uzdevums

Izmantojot 2651. nodaļas datus (nodala2651), salīdziniet ESRI shapefile,
GeoPackage un geoparquet (ja vēlaties arī ESRI File Geodatabase, kuras
uzrakstīšanai var nākties izmantot citu programmu) failu: - aizņemto
diska vietu; - ielasīšanas ātrumu vismaz 10 ielasīšanas izmēģinājumos.

## Faila lejupielāde un izpakošana 

Izveidosim atsevišķus mainīgos, kuros saglabāsim lejupielādes saiti,
lejupielādes ceļu un izpakošanas ceļu:

``` r
url <- "https://data.gov.lv/dati/lv/dataset/40014c0a-90f5-42be-afb2-fe3c4b8adf92/resource/392dfb67-eeeb-43c2-b082-35f9cf986128/download/centra.7z"
lejupielades_cels <- "C:/Users/user/Desktop/HiQBioDiv_macibas/2uzd/centra_virsmezn_dati.7z"
izpakosanas_cels <- "C:/Users/user/Desktop/HiQBioDiv_macibas/2uzd/centra_virsmezn_dati"
```

Turklāt izveidosim arī pašu direktoriju.

``` r
dir.create(izpakosanas_cels, recursive = TRUE) # Ja vēlāma direktorija vēl nav izveidota
```

No atvērto datu portāla lejupielādēsim Centra virsmežniecības datus.

``` r
#Ja pakotne nav instsalēta: install.packages("curl") 
library(curl)

curl_download(url, destfile = lejupielades_cels)

rm(url) #Izdzēst mainīgos, kas vairs nav nepieciešami.
```

Izpakosim failus iepriekš izveidotajā direktorijā:

``` r
#ja pakotne nav instsalēta: install.packages("archive")
library(archive)

archive_extract(lejupielades_cels, dir = izpakosanas_cels)

rm(lejupielades_cels) #Izdzēst mainīgos, kas vairs nav nepieciešami.
```

Ja nepieciešams, pārskatām iegūto failu sarakstu
`list.files(izpakosanas_cels)`.

## Faila nolasīšana un pārveidošana citos formātos

Izveidosim mainīgo, kas satur absolūto ceļu līdz failam. Nolasīsim
failu. Pakotne {sf} automātiski nolasīs visus ar shapefailu saistītos
failus, kas atrodas tajā pašā mapē. Šis process notiek fonā, un nav
jānorāda katra faila ceļš atsevišķi.

Nolasīto shapefile pārveidosim GeoPackage un geoparquet formātos.
Atsevišķos mainīgajos norādīsim ceļus uz katra faila vēlamo izvietošanas
vietu, līdz ar to definējot arī faila nosaukumu.

``` r
#Ja pakotne nav instsalēta: install.packages("sfarrow")
library(sfarrow)

# Pārveidot GeoPackage formātā
geopackage_cels <- "C:/Users/user/Desktop/HiQBioDiv_macibas/2uzd/centra_virsmezn_dati/nodala2651.gpkg"
st_write(nod2651_shapefile, geopackage_cels, driver = "GPKG")
```

    ## Writing layer `nodala2651' to data source 
    ##   `C:/Users/user/Desktop/HiQBioDiv_macibas/2uzd/centra_virsmezn_dati/nodala2651.gpkg' using driver `GPKG'
    ## Writing 91369 features with 70 fields and geometry type Multi Polygon.

``` r
# Pārveidot geoparquet formātā
geoparquet_cels <- "C:/Users/user/Desktop/HiQBioDiv_macibas/2uzd/centra_virsmezn_dati/nodala2651.parquet"
st_write_parquet(nod2651_shapefile, geoparquet_cels)
```

## Formātu salīdzinājums pēc diskā aizņemtās vietas

Apvienosim visus failu ceļus vienā vektorā.

``` r
failu_celi <- c(shapefile_cels, geopackage_cels, geoparquet_cels)
```

Izveidosim datu rāmi, kas satur informāciju par interesējošajiem
failiem. Pārvērsīsim baitus megabaitos un iegūsim pašu failu nosaukumus
(nevis absolūtos ceļus).

``` r
failu_info <- file.info(failu_celi) # Iegūst informāciju par failiem
failu_info$izmers_MB <- failu_info$size / (1024 * 1024) # Baiti uz megabaitiem
failu_info$nosaukums <- basename(rownames(failu_info)) #iegūst faila nosaukumu no pilnā ceļa

rm(failu_celi) #Izdzēst mainīgos, kas vairs nav nepieciešami.
```

Izvadīsim rezultātus konsolē:

``` r
cat(sprintf("Fails: %s\nIzmērs: %.2f MB\n\n", 
            failu_info$nosaukums, failu_info$izmers_MB))
```

    ## Fails: nodala2651.shp
    ## Izmērs: 25.84 MB
    ## 
    ##  Fails: nodala2651.gpkg
    ## Izmērs: 55.12 MB
    ## 
    ##  Fails: nodala2651.parquet
    ## Izmērs: 21.08 MB

**Secinājums**: Vismazāk vietas diskā aizņem *.parquet* fails.

``` r
rm(failu_info) #Izdzēst mainīgos, kas vairs nav nepieciešami.
```

## Formātu salīdzinājums pēc ielasīšanas ātruma

Veiksim 20 ielasīšanas mēģinājumus katram failam, rezultātus reģistrējot
jaunā datu rāmī.

Iegūtais datu rāmis satur kolonnas ar mēģināto funkciju un ielādes
ātrumu nanosekundēs. Saīsināsim katra tipa faila atveršanas funkciju
līdz vienkāršiem paplašinājuma nosaukumiem:

``` r
ielasisanas_atrums$expr <- sub(".*_(.*) <-.*", "\\1", ielasisanas_atrums$expr)
```

Iegūsim kopsavilkumu, atstājot tikai nosaukumu, minimālo, maksimālo un
vidējo vērtību, kā arī izmēģinājumu skaitu.

``` r
rezultati_ielasisanas_atrums <- summary(ielasisanas_atrums)
rezultati_ielasisanas_atrums <- rezultati_ielasisanas_atrums[, c("expr", "min", "max", "mean", "neval")]
```

Izvadīsim rezultātus konsolē:

``` r
cat(sprintf("Faila veids: %s\n Min.: %.2f, Max.: %.2f, Vidējais: %.2f, Mēģinājumi: %.f\n\n", 
            rezultati_ielasisanas_atrums$expr, 
            rezultati_ielasisanas_atrums$min,
            rezultati_ielasisanas_atrums$max,
            rezultati_ielasisanas_atrums$mean,
            rezultati_ielasisanas_atrums$neval
            ))
```

    ## Faila veids: geopackage
    ##  Min.: 3693.45, Max.: 5187.92, Vidējais: 4237.97, Mēģinājumi: 20
    ## 
    ##  Faila veids: geoparquet
    ##  Min.: 720.68, Max.: 1789.68, Vidējais: 1184.01, Mēģinājumi: 20
    ## 
    ##  Faila veids: shapefile
    ##  Min.: 5907.04, Max.: 7695.44, Vidējais: 6650.98, Mēģinājumi: 20

**Secinājums**: Visātrāk tika atvērts *.parquet* fails.

``` r
#Izdzēst mainīgos, kas vairs nav nepieciešami.
rm(rezultati_ielasisanas_atrums, ielasisanas_atrums, nod2651_geopackage, nod2651_geoparquet,
   nod2651_shapefile, shapefile_cels, geopackage_cels, geoparquet_cels
   ) 
```

# 2. uzdevums

Apvienojiet visu Centra virzmežniecību nodaļu datus vienā slānī.
Nodrošiniet, ka visas ģeometrijas ir MULTIPOLYGON, slānis nesatur tukšas
vai nekorektas (invalid) ģeometrijas.

## Jaunās subdirektorijas izveide un failu pārveide

Definēsim un izveidosim atsevišķu subdirektoriju *.parquet* failiem.

``` r
parquet_dir_cels <- "C:/Users/user/Desktop/HiQBioDiv_macibas/2uzd/centra_virsmezn_dati/parquet"
dir.create((parquet_dir_cels), recursive = TRUE)
```

Atradīsim visus shapefailus darba direktorijā, saglabāsim visus
absolūtos ceļus uz shapefailiem vienā vektorā.

``` r
shapefile_visi <- list.files(izpakosanas_cels, pattern = "\\.shp$", full.names = TRUE)
```

Pārveidosim shapefailus *.parquet* formātā un saglabāsim tos
apakšdirektorijā.

## Failu nolasīšana, apvienošana un apstrāde

Ierakstīsim visus absolūtos ceļus uz *.parquet* failiem vienā vektorā.

``` r
parquet_visi <- list.files(parquet_dir_cels, pattern = "\\.parquet$", full.names = TRUE)

rm(parquet_dir_cels, izpakosanas_cels) #Izdzēst mainīgos, kas vairs nav nepieciešami.
```

Nolasīsim un apvienosim visus *.parquet* failus kopā.

``` r
#Ja pakotne nav instsalēta: install.packages("dplyr")
library(dplyr)

# Nolasām un apvienojam visus failus kopā
apvienotais_parquet <- lapply(parquet_visi, st_read_parquet) %>%
  bind_rows()

rm(parquet_visi) #Izdzēst mainīgos, kas vairs nav nepieciešami.
```

Pārveidosim visas ģeometrijas uz MULTIPOLYGON formātu.

``` r
apvienotais_parquet <- apvienotais_parquet %>%
  mutate(geometry = st_cast(geometry, "MULTIPOLYGON"))
```

Pārbaudīsim, vai starp ģeometrijām ir nederīgas, un rezultātus izvadīsim
konsolē. Ja būs atrastas nederīgas ģeometrijas, labosim tās. Veiksim
pārbaudi vēlreiz. Rezultātam jābūt ‘0’.

``` r
# Pārbaudām, vai ir nederīgās ģeometrijas
invalid_geometrijas <- apvienotais_parquet %>% 
  filter(!st_is_valid(geometry))

# Izvadām rezultātu
print(nrow(invalid_geometrijas))
```

    ## [1] 274

``` r
# Ja rezultats > 0, tad labojam
if (nrow(invalid_geometrijas) > 0) {
  apvienotais_parquet <- st_make_valid(apvienotais_parquet)
}
```

Pārbaudīsim, vai ir tukšas ģeometrijas (rezultāts konsolē). Atbildē
saņemsim loģisko vektoru, pēc kura vajadzības gadījumā (ja rezultāts
konsolē \> 0) filtrējam datus (! = otrādi).

``` r
#Pārbaudam vai ir tukšas geometrijas
logika_tuksas_geometrijas <- st_is_empty(apvienotais_parquet)
print(sum(logika_tuksas_geometrijas))
```

    ## [1] 6

``` r
# Filtrējam ārā nederīgās ģeometrijas
apvienotais_parquet <- apvienotais_parquet[!logika_tuksas_geometrijas,]


#Izdzēst mainīgos, kas vairs nav nepieciešami.
rm(invalid_geometrijas, logika_tuksas_geometrijas)
```

# 3. uzdevums

Apvienotajā slānī aprēķiniet priežu (kumulatīvo dažādām sugām) pirmā
stāva šķērslaukumu īpatsvaru, kuru saglabājiet laukā `prop_priedes`.
Laukā `PriezuMezi` ar vērtību “1” atzīmējiet tās mežaudzes, kurās
priedes šķērslaukuma īpatsvars pirmajā stāvā ir vismaz 75% un ar “0” tās
mežaudzes, kurās īpatsvars ir mazāks, pārējos ierakstus atstājot bez
vērtībām. Kāds ir priežu mežaudžu īpatsvars no visām mežaudzēm?

## Apreķins priežu šķerslaukumu īpatsvaru `prop_priedes`

Apreķināsim priežu šķerslaukumu katram ierakstam (kumulatīvo visām pirmā
stāva *Pinus* sugām: (1) priede, (22) ciedru priede, (14) citas
priedes). Atsevišķi apreķināsim arī kopējo šķerslaukumu (visām sugām)
katram ierakstam. Priežu mežaudžu īpatsvaru iegūsim, dalot priežu
šķerslaukumu ar kopējo šķerslaukumu.

``` r
apvienotais_parquet <- apvienotais_parquet %>%
  mutate(
    # Priedes šķērslaukums
    priedes_skerslaukums = (
      ifelse(s10 %in% c("1", "14", "22"), g10, 0) +
        ifelse(s11 %in% c("1", "14", "22"), g11, 0) +
        ifelse(s12 %in% c("1", "14", "22"), g12, 0) +
        ifelse(s13 %in% c("1", "14", "22"), g13, 0) +
        ifelse(s14 %in% c("1", "14", "22"), g14, 0)
    ),
    
    # Kopējais šķērslaukums
    kopejais_skerslaukums = g10 + g11 + g12 + g13 + g14,
    
    # Priedes šķērslaukuma īpatsvars
    prop_priedes = priedes_skerslaukums / kopejais_skerslaukums
  )

# Izdzests starprezultātu kolonnu
apvienotais_parquet$priedes_skerslaukums <- NULL
```

## Priežu mežaudžu noteikšana, kolonnas `PriezuMezi` izveide

Pārbaudīsim vērtību kolonnā `prop_priedes`. Ja priežu šķerslaukuma
īpatsvars ir lielāks vai vienāds ar 0.75, tad šim ierakstam kolonnā
`PriezuMezi` piešķirsim vērtību “1”. Ja kolonnas `prop_priedes` vērtība
ir mazāka par 0.75, kolonnā `PriezuMezi` ievadīsim “0”. Visām pārējām
rindām, kas neatbilst augstāk minētajiem nosacījumiem, kolonnā
`PriezuMezi` paliks bez vērtības.

``` r
apvienotais_parquet <- apvienotais_parquet %>%
  mutate(
    PriezuMezi = case_when(
      prop_priedes >= 0.75 ~ 1,
      prop_priedes < 0.75 & !is.na(prop_priedes) ~ 0,
      TRUE ~ NA_real_
    )
  )
```

## Priežu mežaudžu īpatsvara noteikšana

Lai aprēķinātu priežu mežaudžu īpatsvaru no visām mežaudzēm un atbildētu
uz pēdējo šī uzdevuma jautājumu, izmantosim funkciju mean(). Pārvērsim
rezultātu procentos un izvadīsim rezultātu konsolē.

``` r
priezu_ipatsvars <- mean(apvienotais_parquet$PriezuMezi == 1, na.rm = TRUE)

#Izvadīt rezultātus
cat("Priežu mežaudzes sastāda", round(priezu_ipatsvars * 100, 3), "% no visām mežaudzem")
```

    ## Priežu mežaudzes sastāda 25.296 % no visām mežaudzem

**Atbilde**: Priežu mežaudzes sastāda 25,296% no visām mežaudzēm.

``` r
rm(priezu_ipatsvars) #Izdzēst mainīgos, kas vairs nav nepieciešami.
```

# 4. uzdevums

Apvienotajā slānī, izmantojot informāciju par pirmā stāva koku sugām un
to šķērslaukumiem, veiciet mežaudžu klasifikāciju skujkoku, šaurlapju,
platlapju un jauktu koku mežos. Paskaidrojiet izmantoto pieeju un
izvēlētos robežlielumus. Kāds ir katra veida mežu īpatsvars no visiem
ierakstiem?

## Mežaudžu klasifikācija

Izveidosim sarakstu, kurā sadalīsim koku sugu kodus no Meža valsts
reģistra meža datu
[klasifikatora](https://data.gov.lv/dati/lv/dataset/meza-valsts-registra-meza-dati)
trijos tipos: skujkoki, platlapju koki un šaurlapju koki.

``` r
Koku_tips <- list(
  skujkoki = c(1, 3, 13, 14, 15, 22, 23, 28, 29),
  platlapju_koki = c(10, 11, 17, 18, 24, 25, 26, 27, 32, 35, 50, 61, 63, 64, 65, 66, 67, 69),
  saurlapju_koki =c(4, 6, 8, 9, 12, 16, 19, 20, 21, 62, 68)
  )
```

Galvenajā datu tabulā izveidosim trīs jaunas kolonnas:
`skujkoku_skerslaukums`, `platlapju_skerslaukums`,
`saurlapju_skerslaukums`, kur atradīsies katra koku tipa kumulatīvais
šķerslaukums pirmā stāva kokiem. Veiksim mežaudžu klasifikāciju: katra
pirmā stāva koku tipa kumulatīvo šķerslaukumu dalot ar kopējo
šķerslaukumu, iegūsim koku tipa īpatsvaru. Ja kādam no koku tipiem
īpatsvars būs lielāks vai vienāds ar 0.75, šis tips parādīsies nosaukumā
kolonnā `mezaudzes_veids`. Piemēram: Ja skujkoku īpatsvars pirmā stāva
kokiem ir 0.78, mēs klasificējam šo mežaudzi kā skujkoku mežu. Ja
konkrētā mežaudze nebūs dominējošā koku tipa, tā būs klasificēta kā
“Jaukts mežs”. Gadījumā, ja kopējais šķerslaukums pirmā stāva kokiem būs
vienāds ar “0”, kolonna `mezaudzes_veids` šim ierakstam paliks tukša.

``` r
apvienotais_parquet <- apvienotais_parquet %>%
  mutate(
    # Aprēķinām skujkoku, plātlapju un šaurlapju koku šķērslaukumu
    skujkoku_skerslaukums = 
      ifelse(s10 %in% Koku_tips$skujkoki, g10, 0) + 
      ifelse(s11 %in% Koku_tips$skujkoki, g11, 0) +
      ifelse(s12 %in% Koku_tips$skujkoki, g12, 0) +
      ifelse(s13 %in% Koku_tips$skujkoki, g13, 0) +
      ifelse(s14 %in% Koku_tips$skujkoki, g14, 0),
    platlapju_skerslaukums = 
      ifelse(s10 %in% Koku_tips$platlapju_koki, g10, 0) + 
      ifelse(s11 %in% Koku_tips$platlapju_koki, g11, 0) +
      ifelse(s12 %in% Koku_tips$platlapju_koki, g12, 0) +
      ifelse(s13 %in% Koku_tips$platlapju_koki, g13, 0) +
      ifelse(s14 %in% Koku_tips$platlapju_koki, g14, 0),
    saurlapju_skerslaukums = 
      ifelse(s10 %in% Koku_tips$saurlapju_koki, g10, 0) + 
      ifelse(s11 %in% Koku_tips$saurlapju_koki, g11, 0) +
      ifelse(s12 %in% Koku_tips$saurlapju_koki, g12, 0) +
      ifelse(s13 %in% Koku_tips$saurlapju_koki, g13, 0) +
      ifelse(s14 %in% Koku_tips$saurlapju_koki, g14, 0),
    
    # Veikt klasifikāciju
    mezaudzes_veids = case_when(
      kopejais_skerslaukums == 0 ~ NA_character_,
      skujkoku_skerslaukums / kopejais_skerslaukums >= 0.75 ~ "Skujkoku mežs",
      platlapju_skerslaukums / kopejais_skerslaukums >= 0.75 ~ "Platlapju mežs",
      saurlapju_skerslaukums / kopejais_skerslaukums >= 0.75 ~ "Šaurlapju mežs",
      TRUE ~ "Jaukts mežs"
    )
  )

#Izdzēst mainīgos, kas vairs nav nepieciešami.
rm(Koku_tips)
# Izdzests starprezultātu kolonnas
apvienotais_parquet <- apvienotais_parquet %>%
  select(-skujkoku_skerslaukums, -platlapju_skerslaukums, -saurlapju_skerslaukums, -kopejais_skerslaukums)
```

### Skaidrojums izmantotajai pieejai un robežlielumiem

Slieksnis **0.75** tika izvēlēts, jo tas nodrošina konsekvenci ar esošo
pieeju priežu mežaudžu definēšanā. Šis slieksnis ir piemērots, lai
skaidri noteiktu vai mežaudzē dominē kāds konkrēts koku tips, novēršot
nevajadzīgu skaitļu vairošanu.

Šī pieeja tika izvēlēta, jo kods ir viegli lasāms un nodrošina skaidru
un loģisku secību, kas padara tas struktūru pārskatāmu. Visas kolonnas
tiek izveidotas vienā piegājienā, ļaujot viegli rediģēt un atsevišķi
apstrādāt katru kolonnu, turklāt šī funkcija ir ātrāka nekā, piemēram,
cikls `for`, nodrošinot efektīvāku datu apstrādi.

## Meža veidu īpatsvara noteikšana

Pielietojot funkciju `table()`, uzzinām katra meža veida skaitu. Dalot
to ar kopējo klasificēto mežu skaitu iegūstam īpatsvaru. Apvienojam
rezultātus, noformējam tekstu un izvadām konsolē.

``` r
# Aprēķinām katra meža tipa skaitu
mezaudzes_veidi_skaits <- table(apvienotais_parquet$mezaudzes_veids)

# Aprēķinām katra meža tipa īpatsvaru
mezaudzes_veidi_ipatsvars <- mezaudzes_veidi_skaits / sum(mezaudzes_veidi_skaits)


# Apvienojam rezultātus
mezaudzes_veidi <- data.frame(
  mezaudzes_veids = names(mezaudzes_veidi_skaits),
  n = as.vector(mezaudzes_veidi_skaits),
  meza_veida_ipatsvars = as.vector(mezaudzes_veidi_ipatsvars)
  )

cat(sprintf("Mežaudze veids: %s\n Ierakstu skaits: %.f, Īpatsvars: %.4f\n\n", 
            mezaudzes_veidi$mezaudzes_veids, 
            mezaudzes_veidi$n,
            mezaudzes_veidi$meza_veida_ipatsvars
            )
    )
```

    ## Mežaudze veids: Jaukts mežs
    ##  Ierakstu skaits: 62426, Īpatsvars: 0.1751
    ## 
    ##  Mežaudze veids: Platlapju mežs
    ##  Ierakstu skaits: 2502, Īpatsvars: 0.0070
    ## 
    ##  Mežaudze veids: Skujkoku mežs
    ##  Ierakstu skaits: 148046, Īpatsvars: 0.4152
    ## 
    ##  Mežaudze veids: Šaurlapju mežs
    ##  Ierakstu skaits: 143627, Īpatsvars: 0.4028

**Atbilde**: Katra veida mežu īpatsvars no visiem ierakstiem ir šāds:
Jauktais mežs: 0.1751, Platlapju mežs: 0.0070, Skujkoku mežs: 0.4152,
Šaurlapju mežs: 0.4028.

``` r
#Izdzēst mainīgos, kas vairs nav nepieciešami.
rm(mezaudzes_veidi, mezaudzes_veidi_ipatsvars, mezaudzes_veidi_skaits)
```
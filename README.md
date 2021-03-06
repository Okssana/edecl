

# edecl


Обгортка для API declarations.com.ua

**Встановлення**

Для встановлення з гіт-репозиторію потрібено мати встановлений пакет devtools

    install.packages("devtools")
    devtools::install_github("texty/edecl")
    library(edecl)

Решта вимог мають задовольнитися автоматично при встановленні пакунку. Для зручності ми використовуємо синтаксис pipelines (%>%) та команди бібліотеки dplyr.


**Функції**


  * download_declarations() - завантажує декларації
  * step_to_df() - переводить розділ зі списку декларацій у зручний для обробки даних табличний формат.
  * get_corrected() - виключає зі списку декларації, які були замінені виправленими
  * dintersect() - перетинає дві множини(два списки) декларацій
  * dexclude() - виключає з першого списку декларацій ті, які належать для другого
  * related_companies() - витягає ЄДРПОУ пов'язаних компаній
  * extract_info() - витягує базову інформацію про декларантів (ПІБ, місце роботи, посада, тощо)


**З чим працюємо?**

Для роботи ми використовуємо дані сайту declarations.com.ua. Це сервіс організації “Канцелярська сотня”. На основі декларацій ще старого зразку, а також електронних декларацій сайту НАЗК вони розробили зручний сайт з пошуку інформації про статки посадовців. З декількох причин ми використовуємо саме сервіс “Канцелярської сотні”, а не офіційний сервіс НАЗК:

  * цей сервіс працює надійніше за офіційний, особливо в гарячі періоди, коли всі посадовці будуть подавати декларації;
  * в цьому сервісі реалізований повнотекстовий пошук – наприклад, можна знайти всіх, хто задекларував криптовалюти чи хутряні шуби;
  * видозмінена структура декларацій дозволяє простіше знаходити інформацію зокрема про те, чи є конкретна декларація уточненою, чи ні, а також легко видає усі пов’язані із декларантом фірми;
  * для отримання того ж обсягу даних на declarations.com.ua потрібно робити менше запитів, ніж на офіційний сайт НАЗК;
  * були прецеденти, коли з офіційного сайту пропадали декларації, проте лишалися доступними на сайті “Канцелярської сотні”.

# Приклад застосування
**Отримуємо декларації народних депутатів**

Ініціативи з моніторингу парламенту – такі як ЧЕСНО чи ОПОРА- регулярно укладають рейтинги на основі даних депутатських декларацій. Ми розглянемо, як це можна зробити за допомогою нашого інструменту. 
Почнімо з того, що встановимо наш пакунок, та завантажимо його та бібліотеку dplyr:

    devtools::install_git("https://github.com/texty/edecl/")
    library(edecl)
    library(dplyr)

***download_declarations(), ectract_info()***

Основна складність цього завдання – отримати саме ті декларації які треба. Сайт declarations.com.ua дозволяє шукати декларації за посадою. Нас цікавить посада “народний депутат”. 

    mps2016 <- download_declarations("народний депутат", doc_type = 1, declaration_year = 2016)
    mps_info <- extract_info(mps2016)

Параметр doc_type = 1 означає, що нас цікавлять щорічні декларації,  declaration_year = 2016 -  що декларації за 2016 рік. Якщо б ми хотіли шукати по всій декларації, ми б додали параметр deepsearch = TRUE, але в цьому випадку нам таке не потрібно. 
У змінній mps_info зберігається коротка інформація про декларантів, чиї декларації ми отримали на запит. За допомогою коду

    View(mps_info)

***dexclude()***

ми можемо переглянути зміст цієї таблиці та побачити, що, окрім народних депутатів тут також є багато їхніх помічників. Отже, їх треба виключити зі списку декларацій.


    mps2016 <- mps2016 %>% dexclude(download_declarations("помічник народного депутата", doc_type = 1, declaration_year = 2016))
    mps_info <- extract_info(mps2016)
    View(mps_info)

Функція dexclude отримає в якості аргументів два списки декларацій, і повертає перший список, з якого виключені декларації з другого списку. Таким чином, ми отримаємо результати нашого запиту “народний депутат”, який не містить результатів запиту “помічник народного депутата”. Як і раніше, переглянемо те, що отримали в результаті.
З результатів стає зрозуміло, що виключити тільки помічників – недостатньо. Хоча за української термінології лише депутати Верховної Ради звуться народними депутатами, чимало декларантів роблять помилки і називають себе, наприклад “народними депутатами сільської ради”. Аби виключити місцевих депутатів, застосуємо такий код:

    local_councils <- c(mps_info$id[grepl("міська", mps_info$office)], 
                    mps_info$id[grepl("с?льська", mps_info$office)], 
                    mps_info$id[grepl("районна", mps_info$office)],
                    mps_info$id[grepl("м?ської", mps_info$office)],
                    mps_info$id[grepl("с?льської", mps_info$office)],
                    mps_info$id[grepl("районної", mps_info$office)],
                    mps_info$id[grepl("м?ської", mps_info$position)],
                    mps_info$id[grepl("с?льської", mps_info$position)],
                    mps_info$id[grepl("районної", mps_info$position)],
                    mps_info$id[grepl("селищної", mps_info$office)],
                    mps_info$id[grepl("селищна", mps_info$office)],
                    mps_info$id[grepl("селищної", mps_info$position)],
                    mps_info$id[grepl("селищна", mps_info$position)])
    mps2016 <- mps2016 %>% dexclude(local_councils)
    mps_info <- extract_info(mps2016)
    View(mps_info)

Функція dexclude може в якості другого аргументу приймати як списки декларації, так і просто їхні id. Саме другим варіантом ми і користуємося тут: шукаємо id декларацій, що містять місцеву раду у полі “office” чи “position”, та видаляємо їх.  Ми використовуємо знак питання замість літери “і”, аби убезпечити себе від випадків, коли декларант помилково написав латинську “і” замість української.
Але і отриманий результат буде недосконалим. Якщо придивитися до результатів, то буде зрозуміло, що зайвими є співробітники відділу запитів і звернень народних депутатів Міністерства юстиції, а також місцеві депутати, що завідують “народними домами”. Видалимо й їх. 

    mps2016 <- 
        mps2016 %>%  
        dexclude(download_declarations("народний дім", doc_type = 1, declaration_year = 2016))  %>%
        dexclude(mps_info$id[grepl("юстиції", mps_info$office)])
    mps_info <- extract_info(mps2016)
    View(mps_info)

Нарешті, в деклараціях все-таки лишається півтора більше десятка зайвих осіб, які, утім важко поєднати в одну групу. Просто запишемо їхні id, і видалимо їх.

    mps2016 <-
        mps2016 %>% 
        dexclude(c("nacp_90b1c439-da8c-40a1-ad20-465dcd2fd27e",
                "nacp_c700d733-b4ae-4512-9de5-4f902b5dfedb",
                "nacp_c3059264-5d66-43f4-887a-5d6c7ad40fc9",
                "nacp_f0f86d87-6824-44e3-a3ca-97e18e78d6c5",
                "nacp_a4daa6b4-a26f-4dfe-93df-ef6905deb8eb",
                "nacp_d9d2e527-e4a2-4d2e-a443-802aefeedc09",
                "nacp_54825ebb-109e-4105-b198-7a2e43359c37",
                "nacp_ed608b15-3967-4142-862c-374c9edb6901",
                "nacp_2a10ba58-ed52-470e-b344-1b047c9f7c59",
                "nacp_f3e34f57-fa50-4746-95a9-b0c2ecff2fb0",
                "nacp_b2679c56-1ec3-4d44-ae43-ce27e7224b33",
                "nacp_c41693e2-39e5-4aac-a699-c017ebc8d641",
                "nacp_61ea0794-1b92-4d20-b79e-4fd9c154e16c"))
    mps_info <- extract_info(mps2016)
    View(mps_info)

Якщо ми подивимося кількість унікальних депутатів у нашому списку, то побачимо, що їх 406, хоча народних депутатів 423. 

    length(unique(mps_info$id))

Так трапилося тому, що не всі народні депутати заповнили свою посаду значенням “народний депутат”. Натомість пишуть туди посаду у комітеті. Якщо вивчити список депутатів Верховної Ради та список наших декларантів, то зрозуміємо, що бракує декларацій 15 нардепів. Завантажимо їх, просто зробивши запити за повним ім’ям. 

    additional_mps <- c("Арешонков Володимир Юрійович",
                        "Демчак Руслан Євгенійович",
                        "Дзюблик Павло Володимирович",
                        "Довбенко Михайло Володимирович",
                        "Журжій Андрій Валерійович",
                        "Кривошея Геннадій Григорович",
                        "Ленський Олексій Олексійович",
                        "Луценко Ірина Степанівна",
                        "Ляшко Олег Валерійович",
                        "Папієв Михайло Миколайович",
                        "Парубій Андрій Володимирович",
                        "Рибалка Сергій Вікторович",
                        "Сочка Олександр Олександрович",
                        "Федорук Микола Трохимович",
                        "Фурсін Іван Геннадійович")
    additional_decls <- list()
    for (mp in additional_mps) {
      additional_decls <- c(additional_decls, download_declarations(mp, doc_type = 1, declaration_year = 2016))
    }
    View(extract_info(additional_decls))

Побачимо, що і в цьому списку є зайві декларації – знайшлася одна декларація помічника Ляшка, а запит "Рибалка Сергій Вікторович" повернув декларації, який подав Сергій Рибалко. Видаляємо зайве, і додаємо до загального списку депутатських декларацій.

    additional_decls <- additional_decls %>% 
        			dexclude(c("nacp_0e55db7d-7a40-4612-981d-3cd9775b539e",
                                                        "nacp_62fd7aaf-13e6-46fe-be26-74e7d1bab41c",
                                                        "nacp_61763222-c094-499d-afcd-b150441ab6c4",
                                                        "nacp_930e5c52-e1c7-4734-be5a-a0dc839943e4",
                                                        "nacp_f09a6864-35b6-4e00-888a-bcee1353223e",
                                                        "nacp_ef0fb95a-f962-4b44-a1ff-cbb7c59a33a3"))
    mps2016 <- c(mps2016, additional_decls)
    mps_info <- extract_info(mps2016)

Ще одна проблема – в депутатів буває по дві декларації. Так відбувається тому, що декларант,  якщо зробить помилку, не виправляє дані у старій декларації, а подає нову уточнену. З сайту НАЗК та сайту “Декларації” відомо про те, чи конкретна декларація є уточненою, чи ні, проте невідомо, яку саме декларацію вона уточнює. А ідентичні ПІБ не така вже і велика рідкість.
***get_corrected()***

Функція get_corrected з бібліотеки edecl пропонує рішення проблеми. Отримуючи список декларації, вона знаходить дві, які мають однаковий ПІБ і стосуються того самого року. Серед них вона знаходить ті, які мають хоча б один повністю ідентичний розділ, і з кожної такої пари лишає тільки виправлену, а не виправлену видаляє. Такий алгоритим - не панацея, але найчастіше спрацьовує. 

    mps2016 <- mps2016 %>% get_corrected()
    length(mps2016)

Функція get_corrected, утім, не відловила однієї зайвої декларації депутатки Кацер-Бучковської. Але не через недоліки алгоритму, а просто тому, що депутатка помилися, та неправильно вказала рік у декларації.  Видалимо неправильну декларацію вручну. 

    mps2016 <- mps2016 %>% dexclude("nacp_5e5e4643-1139-46d1-9c1b-123e599b0df6")
    length(mps2016)

Бачимо, що в нас лишилася 421 декларація. Депутати Клюєв, Оніщенко та Заружко декларації не подали, натомість є декларація колишнього депутата Андрія Артеменка. За кількістю все нарешті сходиться. 
Етап правильного відбору декларацій нарешті завершено. Він не такий простий, як хотілося б. З якою групою декларантів ви б не працювали, вам потрібно буде витрачати час на те, аби перевіряти результати запитів, виключати зайві декларації та окремо додавати тих, кого бракує. Проте люди, які мали досвід роботи із “сирим” API НАЗК зрозуміють, що використання бібліотеки все-таки спрощує пошук потрібних даних. 

**Укладаємо рейтинги депутатів**

***step_to_df()***

Найзвичніша форма даних, придатних для аналізу – це таблиця. Але кожна окрема декларація зберігається в ієрархічній формі. Тому перед тим, як укладати рейтинги, треба перевести дані з ієрархічної форми у пласку. 
З цим нам допоможе функція step_to_df. Кожний розділ декларації має свою специфічну структуру, тому і таблички теж створюються під кожний окремий розділ. Номер розділу декларації є другим аргументом у функції. Аби нагадати, за яким номеро зберігаються потрібні вам дані, ви можете знайти на будь-яку декларації на сайті “Канцелярської сотні” чи НАЗК, і подивитися назви розділів. 
Почнімо з того, що знайдемо депутатів із найбільшим доходом (розділ 11) за 2016 рік. 

    top_incomes <- 
      (mps2016 %>% 
      step_to_df(11))$data %>% 
      group_by(fullname) %>% 
      summarise(sum_income = sum(sizeIncome, na.rm = TRUE)) %>% 
      arrange(desc(sum_income))

Функція step_to_df повертає список з 4 табличок. Основна з них – табличка “data”, яка містить інформацію про об’єкти. В другій (“add_rights”) – інформація про додаткові права, які є не в персони, що декларує. Наприклад, якщо депутат орендую квартиру, то в декларації вказуються права оренди депутата на квартиру, а також права власності власника квартири. Права депутата підуть в “data", права власника – в “add_rights”. В третій (“guarantor”) і четвертій (“guarantor_realty”) інформація, актуальна тільки для 13-го розділу декларації про фінансові зобов’язання. Це інформацію про поручителя зобов’язання та про його нерухомість.
Якщо додаткові таблички мають значення, треба при запуску функції step_to_df передати TRUE в параметри add_rights, guarantor та guarantor_realty. Їхнє значення по замовчанню – FALSE, функція працює трішки швидше, але додаткові таблички повертаються порожніми.
Після того, як ми отримали табличку “data”, в нашому коді ми групуємо її за іменем депутата, знаходимо загальний дохід кожного, і сортуємо за спаданням, аби першими йшли депутати, які отримали найбільший дохід.
Цей результат враховує доходи кожного депутата і членів їхній сімей. Якщо ми хочемо подивитися тільки власне депутатські доходи, треба фільтрувати за параметром person. Якщо він дорівню “1”, значить, це доходи декларанта, а не членів його сімей. 

    top_incomes <- 
      (mps2016 %>% 
      step_to_df(11))$data %>% 
      filter(person == "1") %>% 
      group_by(fullname) %>% 
      summarise(sum_income = sum(sizeIncome, na.rm = TRUE)) %>% 
      arrange(desc(sum_income))

Поле ownershipType відповідає за тип права на об’єкт. Для доходів це не дуже актуально, але коли йдеться про об’єкти нерухомсті, зрозуміло, що можна володіти часткою об’єкту чи орендувати його. Тому, якщо ми захочемо подивитися, якою площею землі володіють депутати, нам треба враховувати поля ownershipType та percent.ownership. 

    land_owners <- 
      (mps2016 %>% 
      step_to_df(3))$data %>% 
      filter(objectType == "Земельна ділянка", ownershipType == "Власність") %>% 
      mutate(areaPart = totalArea * (percent.ownership / 100)) %>% 
      group_by(fullname) %>% 
      summarise(sumArea = sum(areaPart)) %>% 
      arrange(desc(sumArea))

Групувати можна не лише за депутатами. Наприклад, можна побачити, в якому районі більшість депутатів полюбляють жити. Для цього треба виділити квартири і житлові будинки, згрупувати за поштовим індексом та порахувати кількість об’єктів нерухомості за кожним з індексів. 

    postcodes <- 
      (mps2016 %>% 
      step_to_df(3))$data %>% 
      filter(objectType == "Квартира" | objectType == "Житловий будинок") %>% 
      group_by(ua_postCode) %>% 
      summarise(appartments_houses_number = n()) %>%
      arrange(desc(appartments_houses_number))
    View(postcodes)

Побачимо, що на перших місцях за популярністю серед депутатів – центральні райони Києва і Львова, а також м. Козин (індекс 08711), що також відоме як місце елітної нерухомості.
Також журналістів часто цікавить інформація про автомобілі народних депутатів. Тут, утім, не все так просто. Депутати вказують марку і моделі машин самостійно, а отже, та само марка чи модель може бути записана у десяток різних способів (“Mercedes-Benz”, “Mercedes”, “Mersedes”, “Мерседес” і т.ін.). Тому єдине, що можна швидко порахувати із впевненістю – загальна кількість автомобілів. 

    autos <-
      (mps2016 %>% 
      step_to_df(6))$data %>% 
      filter(objectType == "Автомобіль легковий") %>% 
      group_by(fullname) %>% 
      summarise(n_autos = n()) %>% 
      arrange(desc(n_autos))

Якщо ж хочеться все-таки подивитися на конкретні марки і моделі, то доведеться експортувати дані декларацій у формат .csv:

    autos <-
      (mps2016 %>% 
      step_to_df(6))$data %>% 
      filter(objectType == "Автомобіль легковий") %>% 
      write.csv("mps_autos.csv", row.names = FALSE)

Бренди і моделі авто, які експортовані у цей файл, буде відносно легко почистити програмою OpenRefine. Після того аналіз можна робити у R, або ж у звичайній зведеній таблиці. 

**Як працювати далі?**

Кожний розділ декларацію має свою специфічну структуру, розписувати її було б надто довго. Кожен розділ містить стовпчик “person”, в якому “1” позначає декларанта, а інші значення – коди членів його сім’ї. Більшість розділів містить стовпчики ownershipType та percent.ownership, які дозволяють побачити право декларанта на об’єкти. Але все одно доведеться придивлятися до назв стопчиків та їхнього змісту, аби розуміти можливі історії, які приховує кожен з розділів. 
Також не забувайте, що про помічені помилки та свої побажання ви можете лишати у розділі Issues репозиторію бібліотеки на GitHub. 






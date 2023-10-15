---
layout: post
title: 'Тестируем Rails c помощью Ferrum'
date: 2023-10-15 10:13:04 +0300
categories: Rails7
---

# Rails7. Ferrum для тестов приложения.

Много тестов - это хорошо, особенно если они еще и по делу.
Но много тестов - это время, которое они занимают в процессе выполнения. И если его можно уменьшить - почему бы это не сделать.

Вдохновившись описанными получаемыми результатами:

-   `Отметим что Ferrum работает в 5-10 раз быстрее чем другие фреймворки.` [Тестирование пользовательских сценариев с помощью Ferrum](https://habr.com/ru/companies/bimeister/articles/749122/)
-   `Our tests are much faster` [Migrating Selenium system tests to Cuprite ](https://dev.to/nejremeslnici/migrating-selenium-system-tests-to-cuprite-42ah)
-   `Ferrum is faster than Selenium` - [Open-sourcing Ferrum: a fearless Ruby Chrome driver](https://evrone.com/blog/ferrum-ruby-chrome-driver)

, пробуем использовать ferrum вместо "медленных" selenium-webdrivers.
Чтобы получить относительно сопоставимые результаты, сделам отдельный MR и сравним, что получится ДО и ПОСЛЕ.

Количество и качество тестов - одинаковое в данном случае.

## Настройки

Как и описано, например вот здесь - [Proper browser testing in Ruby on Rails (evil martians)](https://evilmartians.com/chronicles/system-of-a-test-setting-up-end-to-end-rails-testing), настройки не представляют большой сложности.

```ruby
Gemfile
...
+  gem 'cuprite'
-  gem 'webdrivers'
...
```

Конфигурация драйвера

```ruby
Capybara.server = :puma, { Silent: true }
Capybara.register_driver :cuprite do |app|
  # FIXME: Two different keys for headless mode exist - one for Capybara (Cuprit) and one for Chromium - headless mode
  # [Chrome's headless mode](https://developer.chrome.com/articles/new-headless/#new-headless-in-selenium-webdriver)
  # It's impossible to set two keys with the same values - does not work

  if ENV['HEADLESS'].in?(%w[n 0 no false])
    browser_options = { 'no-sandbox': nil }
    headless = false
  else
    browser_options = { 'no-sandbox': nil, headless: 'new' }
    headless = 'new'
  end
  options = {
    browser_options:,
    window_size: [1600, 1200],
    inspector: true,
    headless:
    # js_errors: true
    # slowmo: 2,
    # process_timeout: 15
  }

  Capybara::Cuprite::Driver.new(app, options)
end

Capybara.save_path = DownloadHelpers::DOWNLOAD_PATH || './tmp/capybara'
# disable CSS transition and jQuery animation
Capybara.disable_animation = true
# Usually, especially when using Selenium, developers tend to increase the max wait time.
# With Cuprite, there is no need for that.
# We use a Capybara default value here explicitly.
# Capybara.default_max_wait_time = 2

# Normalize whitespaces when using `has_text?` and similar matchers,
# i.e., ignore newlines, trailing spaces, etc.
# That makes tests less dependent on slightly UI changes.
Capybara.default_normalize_ws = true

Capybara.javascript_driver = :cuprite


```

## Запуск. Ошибки

Как и описано в упомянутых источниках и в ссылках внизу, часть тестов завершилась с ошибками.

Ошибки поделились на несколько категорий.

### Самая распространенная:

-   перестают работать одни из самых используемых методов - click_link (и его разновидности, например find('#id_blabla').click).

Причина оказалась простая и отчасти уважительная, драйвер "пытается" детально разобраться в структуре линка и, обнаруживая в нем что-то еще, выдает ошибку.
Это "что-то еще" оказалось используемыми css наподобие ::before, используемыми bootstrap и bulma для, например, смены background строки таблицы при наведении.

Везде, где применяются еще какие то дополнительные анимационные css, метод click_link не будет работать.

Пользуясь подсказкой ferrum (или cuprite), меняем click на trigger('click'), игнорируя предупреждение о том, что возможно нажимаем не совсем в той области, где планировали.

Обнаруживается вторая ошибка - метод trigger('click') на "нормальных" без css линках не работает.

Делаем два разных шага в cucumber для устранения данных ошибок, разбираемся, где линки с css, а где нет - и в результате большая часть (примерно 90%) тестов начинает работать с новым драйвером.

### Ошибка два:

Часть тестов, использующих прямой вызов JS для того, чтобы установить или получить значение из применяемых компонент, не работает.
Устанавливаемые значения - все срабатывает, а вот возвращается всегда nil.

В описании Ferrum [JavaScript](https://github.com/rubycdp/ferrum#javascript) находим, что для выполнения JS и выполнения с возвратом значения используются разные методы (evaluate/execute).
Возможно при реализации Cuprite этот нюанс не был учтен, меняем метод там, где необходимы значения и тесты начинают работать.

### Ошибка три:

Вызвана тем, что headless и headful режим работает различно. Проявляется это просто - в режиме headful тест выполняется, в режиме headless - падает с ошибкой, связанной с тем, что не отрабатывает стандартная функция onchange: 'this.form.submit()'.

Руководствуясь документом от Google [Chrome’s Headless mode](https://developer.chrome.com/articles/new-headless/) меняем headless mode на новую.
Здесь возникает еще один не очень понятный момент, связанный с тем, что mode определяется в ДВУХ местах. В настройках параметров драйвера и в настройках параметров браузера драйвера. В итоге пока оставляем так, как приведено в конфигурации.

И после этого все тесты начинают работать.

## Результаты

Это не benchmark, это просто вывод времени выполнения каждого теста для одной и той же группы тестов, на одной и той же локальной машине.

Исходные данные:

-   Cucumber 70 scenarios (550 steps)
-   RSpec 346 examples

Сравнительная таблица

| Driver            | RSpec                  | Cucumber  |
| ----------------- | ---------------------- | --------- |
| Selenium          | 1 minute 41.55 seconds | 2m31.672s |
| Cuprite           | 2 minutes 26.6 seconds | 3m47.793s |
| Cuprite(Chromium) | 2 minutes 17.7 seconds | 3m14.874s |

Результаты, мягко говоря, неожиданные. Поэтому на всякий случай поменяли браузер c Chrome на Chromium - что не сильно помогло.
Близкие результаты Rspec определяются тем, что тестов с capybara там немного.
А вот для cucumber - половина тестов использует capybara.

## Еще немного о странностях

Дальнейшее изучение проблемы с быстродействием драйвера ferrum привело к совершенно неожиданным результатам.

Одни и те же тесты в одном и том же окружении при смене версии браузера ведут себя совершенно по разному.
Для примера:

Версия 108.0.5359.98 (Официальная сборка), (64 бит) - у коллеги на машине, linux

-   Падает один тест, который до этого не падал. Ошибка связана с тем, что не работает функция onclick. Возможно в этой версии браузера headless еще "старой" конструкции.

Версия 114.0.5735.198

-   все тесты выполняются без ошибок.

Google Chrome 115.0.5790.102 - последняя на текущий момент.

-   начали падать ДВА теста. Стабильно. На разных (двух) машинах, с MacOS и Linux.
    При этом попытка "разобраться", в чем там дело, завершилась несколько неудачно.
    Эти же тесты, но запушенные по ОДНОМУ - не падают вообще.

## Предварительные выводы.

Плюсы:

-   Возможность использования дополнительных инструментов для позиционирования, заполнения, анализа страницы
-   Отладка - возможность debug с текущей страницей - это просто замечательно.
-   Большое кол-во дополнительных полезных функций, как то ожидание ответа по AJAX, поиск по css и т.д.

Минусы:

-   Поведение с разными версиями браузера ???
-   Быстродействие ???

Пока полученные результаты не позволяют сделать вывод о целесообразности замены selenium-webdrivers на ferrum.

Вполне возможно, что описываемые результаты увеличения быстродействия в разы связаны с какими то иными причинами.
Возможно проблема в том, что именно тестируется с помощью capybara.

Используемые нами тесты - это стандартные интеграционные тесты и не включают тестирование интерфейса как такового.
Активно используется поиск и заполнение полей, чек-боксов, выбор из списков, управление JS компонентами типа редактор tinyMCE и т.д.
Все тестируемые компоненты форм требуют авторизации, здесь используется стандартный helper от devise и в случае selenium-webdrivers, и в случае ferrum.
Вполне возможно, что быстродействие можно будет увеличить за счет использования предоставляемого механизма управления сессиями и отказа от использования sign-in.
Но это уже несколько иная ситуация, тем более, что при наличии различных ролей проблема не будет решена.
А проводить тестирование под условным "суперпользователем" как то не очень логично, особенно, если тестируются ограничения этого доступа.

Полученные результаты не позволяют сделать вывод о целесообразности замены используемого драйвера selenium-webdrivers на ferrum по состоянию разработки на текущий момент.

-   ferrum (0.13)
-   cuprite (0.14.3)
-   Rails 7.0.6

## Используемые источники

-   [cuprite](https://github.com/rubycdp/cuprite)
-   [ferrum](https://github.com/rubycdp/ferrum)
-   [Proper browser testing in Ruby on Rails (evil martians)](https://evilmartians.com/chronicles/system-of-a-test-setting-up-end-to-end-rails-testing)
-   [Modern Ruby Web Automation and Scraping with Ferrum](https://dev.to/libsyz/modern-ruby-web-automation-and-scraping-with-ferrum-10dh)
-   [How to implement Cuprite into Capybara](https://marsbased.com/blog/2022/04/18/cuprite-driver-capybara-sample-implementation/)
-   [Module: Ferrum](https://www.rubydoc.info/gems/ferrum/0.2.1/Ferrum)
-   [Migrating Selenium system tests to Cuprite ](https://dev.to/nejremeslnici/migrating-selenium-system-tests-to-cuprite-42ah)

---
title: Изменение секретов .env файла
draft: false
tags:
  - CI
  - Linux
---
 ### Использование `sed`
 
 Для использования секретов, указанных в `.env `файле, при выполнении **workflow** можно скопировать файл с примерами `.env.example`  и, используя команду `sed`, изменить значения переменных, указанных в качестве примера, на секреты, указанные в GitHub Secrets репозитория.

```GitHub
  backend_tests:
    name: Backend Tests
    runs-on: ubuntu-latest
    needs: linter
    steps:
      - name: Check out repository
        uses: actions/checkout@v4
  
      - name: Add secrets to.env
        run: |
          cp -r backend/.env.example backend/.env
          sed -i "/^VARIABLE=/s/=.*/=${{ secrets.VARIABLE }}/" backend/.env
```

Команда `sed` — это потоковый редактор текста (**s**tream **ed**itor), работающий по принципу замены. Его можно использовать для поиска, вставки, замены и удаления фрагментов в файле.
* `-i` — edit in-place;
* `/^VARIABLE=/` — ищет строки, содержащую указанную переменную;
* `s` — команда замещения;
* `/=.*/ `— заменить все, что идет после знака `=` ;
* `=${{ secrets.VARIABLE }}` — заменить на секрет из GitHub Secrets;
* `backend/.env` — файл, в котором надо выполнить операцию.

### Синтаксис `s`-команды
```
s/AAAA/BBBB/options
```
* `s` — команда замещения;
* `AAAA`— [[Регулярные выражения|regex]] , который необходимо заменить;
* `BBBB`— regex , на который необходимо заменить значение `AAAA`;
----
📂 [[GitHub Actions]]

Последнее изменение: 23.09.2024 15:16
=====О приложении=====
Приложение представляет собой микросервис аутентификации (наверняка, ещё не финальная версия) для итогового проекта 'Интернет-магазин "Закусвилл"'.
Т.к. он как раз работат с пользователями, а я всё равно начинал его делать для проекта, то решил, что его можно как раз заиспользовать для данной домашки, чтобы не пустую болванку делать, а уже что-то полезное и более интересное.

Рабочая директория для установки приложения:
cd <homework4_dir_path>

=====Установка=====
Быстрая инструкция:
kubectl apply -f ./infra/secrets-infra.yaml
helm install --namespace=homework otus-zv-postgres oci://registry-1.docker.io/bitnamicharts/postgresql -f ./infra/pg-values.yaml
kubectl apply -f job-pg-init.yaml,secrets-common.yaml,secrets-auth.yaml,cm-common.yaml,dp-auth.yaml,service-auth.yaml,ingress-auth.yaml

---
Подробная инструкция:
1) поставить секреты, которые нужны для чарта bitnami postgres
kubectl apply -f ./infra/secrets-infra.yaml

2) поставить постгрю с использованием секретов
helm install --namespace=homework otus-zv-postgres oci://registry-1.docker.io/bitnamicharts/postgresql -f ./infra/pg-values.yaml

3) выполнить job по db init
kubectl apply -f ./job-pg-init.yaml

4) ставим секреты и конфигмапы для приложения(й):
kubectl apply -f secrets-common.yaml,secrets-auth.yaml,cm-common.yaml

5) ставим приложение
kubectl apply -f dp-auth.yaml,service-auth.yaml,ingress-auth.yaml

Джоба job-pg-init.yaml идемпотентна и безопасно создаёт БД, схему БД для сервиса, конкретного пользователя сервиса и назначает права владельца на схему (не на БД целиком) с правом коннекта к БД.
Остальную структуру таблиц для AspNetCoreIdentity + __EFMigrationHistory (если они не существуют) сервис создаёт в выделенной схеме сам, накатывая Code-First миграции на старте приложения, после чего приложение стартует и готово к работе.

=====Удаление=====
helm uninstall --namespace=homework otus-zv-postgres
kubectl delete pvc data-otus-zv-postgres-postgresql-0
kubectl delete -f ./infra/secrets-infra.yaml,ingress-auth.yaml,service-auth.yaml,dp-auth.yaml,cm-common.yaml,secrets-auth.yaml,secrets-common.yaml

=====Работа с приложением====
Swagger в браузере: http://otus.zv.auth.arch.homework/swagger/index.html
Эндпоинт хелсчека: http://otus.zv.auth.arch.homework/health

Flow методов такой получается:
1) Регистрация нового пользователя.
2) Логин пользователя (получение JWT). Для дальнейших запросов потребуется добавить заголовок Authentication: Bearer <jwt_token>
3) Проверка аутентификации
4) Получение профиля пользователя
5) Изменение данных профиля пользователя
6) Удаление пользователя

Соответствующая коллекция постмана прилагается.
При работе в Postman сначала нужно добавить переменную окружения постмана zakusvill_auth_base_uri со значением http://otus.zv.auth.arch.homework
После успешного выполнения метода логина не забыть полученный токен прописать в переменную окружения zakusvill_auth_bearer_token.
В случае работы через сваггер для добавления токена во все исходящие запросы нажать кнопку "Authorize" в правом верхнем углу, вставить в поле ввода jwt-токен и нажать "Authorize" в попапе.
После этого можно выполнять остальные 4 метода.

---
Image собран из исходников командой
docker build --progress=plain -t aquirier/otus-zv-auth:homework -f ./Otus.Zakusvill/Otus.Zakusvill.AuthApi/Dockerfile .
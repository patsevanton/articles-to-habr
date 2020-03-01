В этом посте рассмотрим хранилище rpm артефактов c помощью простого скрипта с inotify + createrepo. Заливка артефактов осуществляется через webdav + apache. Почему не webdav + nginx будет написано в конце.

Итак, решение должно отвечать cледующим требованиям для организации только RPM хранилища:

- бесплатное
- простое в установке и обслуживании.

Почему не [SonaType Nexus](https://habr.com/ru/post/473358/) или [Pulp](https://pulpproject.org/):

- Хранение в [SonaType Nexus](https://habr.com/ru/post/473358/) или [Pulp](https://pulpproject.org/) многих типов артефактов приводит к тому что [SonaType Nexus](https://habr.com/ru/post/473358/) или [Pulp](https://pulpproject.org/) становятся единой точкой отказа.
- Высокая доступность (high availability) в [SonaType Nexus](https://habr.com/ru/post/473358/) является платной.
- [Pulp](https://pulpproject.org/) мне кажетя переусложенным решением.
- Артефакты в [SonaType Nexus](https://habr.com/ru/post/473358/) хранятся в blob. При ввнезапном выключении электричества вы не сможете восстановить blob, если у в вас нет бекапа.


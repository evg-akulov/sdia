# Платёжная вертикаль банка

[Текстовая секция и диаграммы "Контекст, границы и коммуникации"](description/description_1.md)

---

[Текстовая секция и диаграммы "Данные и консистентность"](description/description_2.md)

---

* [Текстовая секция "Надежность, масштабирование и наблюдаемость"](description/description_3.md)
* [Sequence диаграмма - сценарий "провайдер тормозит/падает"](sequence/sequence_provider.puml)
* [C4 Container-диаграмма Основные топики](c4/container_topics.puml)
* [Sequence-диаграмма сценария: "сообщение не удалось обработать несколько раз -> попадание в DLQ -> дальнейшая ручная/отложенная обработка"](sequence/sequence_dlq.puml)
* [C4 диаграмма Cache](c4/container_cache.puml)
* [Sequence-диаграмма сценария: "клиент открывает историю платежей"](sequence/sequence_cache.puml)
* [C4 диаграмма Observability](c4/container_observability.puml)

---

* [C4 Container-диаграмма AuthN/AuthZ](c4/container_security.puml)
* [Sequence-диаграмма_OIDC Authorization Code Flow](sequence/sequence_OIDC_flow.puml)
* [security-auth.md](description/security-auth.md)
* [C4 Container-диаграмма_Webhook security](c4/container_webhook_security.puml)
* [Sequence-диаграмма_Webhook validation + anti-replay + idempotency](sequence/sequence_webhook_validation.puml)
* [webhooks-security.md](description/webhooks-security.md)
* [C4 Container-диаграмма_Vault + injection](c4/container_vault.puml)
* [Sequence-диаграмма_Vault agent + rotation](sequence/sequence_vault.puml)
* [secrets-management.md](description/secrets-management.md)
* [C4 Container-диаграмма_CI/CD + Argo Rollouts + Gateway](c4/container_cicd_argo_rollouts.puml)
* [Sequence-диаграмма_canary/blue-green rollout + rollback](sequence/sequence_canaryrollout.puml)
* [пример .gitlab-ci.yml (с этапами)](config/.gitlab-ci.yml)
* [пример Rollout YAML (canary)](config/rollout.yaml)
* [deployment-strategy.md](description/deployment-strategy.md)
* [C4 Container-диаграмма_feature toggle migration](c4/container_db_migration_feature_toggle.puml)
* [Sequence-диаграмма_expand/contract migration](sequence/sequence_db_migration_expand.puml)
* [db-migration-feature-toggle.md](description/db-migration-feature-toggle.md)
* [release-checklist.md](description/release-checklist.md)
* [runbook.md](description/runbook.md)
---

Курс system design in action, @2025
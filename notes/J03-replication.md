# Que se passe-t-il si un broker tombe ?

Dans ce projet, les topics créés avec les paramètres par défaut ont trois répliques par partition (`KAFKA_DEFAULT_REPLICATION_FACTOR: 3`) et `KAFKA_MIN_INSYNC_REPLICAS: 2`. Les trois brokers participent également au quorum de contrôleurs KRaft. Ces valeurs par défaut peuvent être remplacées par la configuration d’un topic particulier.

## Panne d’un broker

Si le broker en panne était le **leader** d’une partition, les brokers restants détectent son absence. Kafka élit un nouveau leader parmi les répliques synchronisées (**ISR**, *in-sync replicas*). Les producteurs et consommateurs peuvent subir une courte interruption pendant la détection et l’élection ; les clients Kafka réessaient normalement leurs requêtes. Une fois l’élection terminée, la partition redevient disponible via son nouveau leader.

Si le broker en panne n’héberge que des **followers** pour une partition, son leader continue à servir cette partition. La réplication de cette partition est temporairement réduite pendant l’indisponibilité.

Avec un facteur de réplication de 3, une seule panne laisse normalement deux copies disponibles. Si les deux restantes sont synchronisées, elles satisfont `min.insync.replicas=2` : les écritures peuvent continuer **si le producteur demande `acks=all`**. Ce réglage côté producteur est important : `min.insync.replicas` ne force pas à lui seul `acks=all`.

Après redémarrage, le broker récupère les données manquantes auprès du leader et ses répliques rejoignent de nouveau l’ISR lorsqu’elles sont suffisamment à jour. Kafka peut alors rétablir le facteur de réplication nominal.

## Si les répliques synchronisées deviennent insuffisantes

Si une partition a moins de deux répliques dans l’ISR, les écritures avec `acks=all` échouent plutôt que d’être acquittées avec un niveau de réplication inférieur au minimum demandé. Le producteur reçoit une erreur de type `NotEnoughReplicas` et doit réessayer ou signaler l’échec à l’application. Cette règle privilégie la durabilité des écritures au détriment de leur disponibilité temporaire.

La lecture dépend de l’existence d’un leader disponible pour la partition et des réglages des consommateurs. En cas d’indisponibilité ou de changement de leader, les lectures peuvent être brièvement interrompues. La réplication protège contre la panne d’un broker, mais ne remplace ni les sauvegardes ni la surveillance.

## Panne du quorum de contrôleurs KRaft

Les trois brokers configurés ont aussi le rôle `controller` et sont membres du quorum KRaft. Un quorum de trois membres exige une majorité, donc au moins **deux contrôleurs disponibles**. La panne d’un broker laisse cette majorité : le cluster peut continuer à gérer son état et ses élections. Si deux brokers tombent, le quorum est perdu ; les changements de métadonnées et les élections ne peuvent plus être assurés normalement, ce qui limite fortement la disponibilité du cluster, même si certaines données de partitions restent présentes sur le broker survivant.

## En résumé

- **Un broker tombe :** Kafka élit si nécessaire un nouveau leader depuis l’ISR ; une courte interruption est possible, puis les répliques se resynchronisent au retour du broker.
- **Deux brokers tombent :** le minimum d’ISR pour les écritures `acks=all` n’est généralement plus atteint, et le quorum KRaft de trois nœuds est perdu.
- Pour bénéficier du compromis de durabilité configuré, le producteur doit notamment utiliser `acks=all`. Le facteur de réplication et `min.insync.replicas` sont des réglages distincts, et les paramètres effectifs du topic doivent être vérifiés.

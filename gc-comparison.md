# Comparaison G1GC vs ZGC vs Parallel GC

## Introduction

Le choix du Garbage Collector (GC) est crucial pour les performances d'une application Java. Chaque collecteur a ses propres caractéristiques, avantages et cas d'usage optimaux. Ce guide compare en détail les trois collecteurs les plus utilisés en production.

## Vue d'ensemble des Collecteurs

### Parallel GC (ParallelGC)
- **Type** : Collecteur générationnel stop-the-world
- **Disponibilité** : Depuis Java 1.4, par défaut jusqu'à Java 8
- **Philosophie** : Maximiser le débit (throughput) au détriment de la latence

### G1GC (Garbage First)
- **Type** : Collecteur générationnel à faible latence
- **Disponibilité** : Depuis Java 7, par défaut depuis Java 9
- **Philosophie** : Équilibre entre débit et latence avec des pauses prévisibles

### ZGC (Z Garbage Collector)
- **Type** : Collecteur concurrent à très faible latence
- **Disponibilité** : Depuis Java 11 (expérimental), production ready depuis Java 17
- **Philosophie** : Latence ultra-faible indépendamment de la taille du heap

## Comparaison Détaillée

### 1. Architecture et Fonctionnement

#### Parallel GC
```
┌─────────────────────────────────────┐
│           Heap Memory               │
├─────────────────┬───────────────────┤
│   Young Gen     │    Old Gen        │
├─────┬───────────┼───────────────────┤
│Eden │ Survivor  │   Tenured Space   │
└─────┴───────────┴───────────────────┘
```

**Caractéristiques :**
- Division classique Young/Old Generation
- Collection parallèle avec tous les threads disponibles
- Stop-the-world complet pendant la collection
- Algorithme mark-sweep-compact pour l'Old Generation

#### G1GC
```
┌─────────────────────────────────────┐
│          Heap divisé en régions     │
├───┬───┬───┬───┬───┬───┬───┬───┬───┤
│ E │ S │ O │ E │ S │ O │ H │ E │ O │
└───┴───┴───┴───┴───┴───┴───┴───┴───┘
E = Eden, S = Survivor, O = Old, H = Humongous
```

**Caractéristiques :**
- Heap divisé en régions de taille fixe (1MB à 32MB)
- Collection incrémentale avec pause-time goals
- Concurrent marking pour l'Old Generation
- Evacuation pause pour le compactage

#### ZGC
```
┌─────────────────────────────────────┐
│     Heap avec Colored Pointers      │
├─────────────────────────────────────┤
│  Objects + Metadata dans pointeurs │
│     Collection concurrente          │
└─────────────────────────────────────┘
```

**Caractéristiques :**
- Non-generational par défaut (generational disponible depuis Java 21)
- Colored pointers pour le tracking d'objets
- Collection entièrement concurrente
- Load barriers pour la cohérence

### 2. Performance et Latence

| Métrique | Parallel GC | G1GC | ZGC |
|----------|-------------|------|-----|
| **Débit (Throughput)** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Latence moyenne** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Pauses max** | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Prédictibilité** | ⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Évolutivité heap** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### 3. Consommation de Ressources

#### CPU Overhead
- **Parallel GC** : Faible pendant l'exécution, pics élevés pendant GC
- **G1GC** : Modéré et constant, travail concurrent
- **ZGC** : Plus élevé de base, mais très stable

#### Mémoire Overhead
- **Parallel GC** : ~2-5% du heap
- **G1GC** : ~5-10% du heap (remembered sets, card tables)
- **ZGC** : ~10-20% du heap (colored pointers, metadata)

## Configuration et Tuning

### Parallel GC
```bash
# Configuration basique
-XX:+UseParallelGC

# Tuning avancé
-XX:MaxGCPauseMillis=200
-XX:GCTimeRatio=19
-XX:NewRatio=2
-XX:SurvivorRatio=8

# Monitoring (Java 9+)
-Xlog:gc*:gc.log:time
# Ou pour Java 8
-XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps
```

### G1GC
```bash
# Configuration basique
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# Tuning avancé
-XX:G1HeapRegionSize=16m
-XX:G1NewSizePercent=20
-XX:G1MaxNewSizePercent=40
-XX:G1MixedGCCountTarget=8
-XX:G1MixedGCLiveThresholdPercent=85

# Monitoring (Java 9+)
-Xlog:gc*:gc.log:time
# Ou pour Java 8
-XX:+PrintGC -XX:+PrintGCDetails
```

### ZGC
```bash
# Configuration basique
-XX:+UseZGC
# Note: -XX:+UnlockExperimentalVMOptions plus nécessaire depuis Java 17

# Tuning avancé
-XX:SoftMaxHeapSize=30g
-Xlog:gc*:gc.log:time

# Java 21+ - ZGC Generational (expérimental)
-XX:+UseZGC -XX:+UnlockExperimentalVMOptions -XX:+UseZGenerationalGC
```

## Cas d'Usage Recommandés

### Utilisez Parallel GC quand :
✅ **Le débit est prioritaire sur la latence**
- Applications batch
- Traitement de données massives
- Calculs scientifiques
- ETL processes

✅ **Heap de taille modérée (< 4GB)**

✅ **Pauses GC acceptables (> 100ms)**

### Utilisez G1GC quand :
✅ **Équilibre débit/latence requis**
- Applications web avec SLA modérés
- Services métier avec contraintes de latence
- Applications avec heap 4GB-32GB

✅ **Pauses prévisibles importantes**
- SLA avec contraintes de latence (< 100ms)
- Applications interactives

✅ **Heap de grande taille avec allocation mixed**

### Utilisez ZGC quand :
✅ **Latence ultra-faible critique**
- Trading systems
- Applications temps réel
- Gaming servers
- Applications haute fréquence

✅ **Très gros heap (> 32GB)**
- Applications Big Data
- Caches en mémoire massifs
- Analytics en temps réel

✅ **Pauses < 10ms indispensables**

## Métriques de Performance

### Métriques clés à surveiller

```java
// Exemple de monitoring JVM
public class GCMetrics {
    
    @EventListener
    public void handleGCEvent(GarbageCollectionNotificationInfo info) {
        String gcName = info.getGcName();
        long duration = info.getGcInfo().getDuration();
        long beforeUsed = info.getGcInfo().getMemoryUsageBeforeGc()
            .values().stream().mapToLong(MemoryUsage::getUsed).sum();
        long afterUsed = info.getGcInfo().getMemoryUsageAfterGc()
            .values().stream().mapToLong(MemoryUsage::getUsed).sum();
        
        logger.info("GC {} - Duration: {}ms, Freed: {}MB", 
                   gcName, duration, (beforeUsed - afterUsed) / 1024 / 1024);
    }
}
```

### Seuils d'alerte recommandés

| Collecteur | Pause Max | Fréquence | Throughput |
|------------|-----------|-----------|------------|
| **Parallel GC** | < 1000ms | < 5% du temps | > 95% |
| **G1GC** | < 200ms | < 5% du temps | > 90% |
| **ZGC** | < 10ms | Toujours | > 85% |

## Migration entre Collecteurs

### De Parallel GC vers G1GC
```bash
# Avant
-XX:+UseParallelGC -Xmx8g

# Après - Migration progressive
-XX:+UseG1GC -Xmx8g -XX:MaxGCPauseMillis=200

# Monitoring pendant 1-2 semaines
# Ajustement si nécessaire
-XX:G1HeapRegionSize=16m
```

### De G1GC vers ZGC
```bash
# Avant
-XX:+UseG1GC -Xmx32g -XX:MaxGCPauseMillis=100

# Après - Test en staging d'abord
-XX:+UseZGC -Xmx32g
-XX:SoftMaxHeapSize=30g

# Monitoring intensif requis
```

## Troubleshooting Commun

### Parallel GC - Long pauses
```bash
# Diagnostics (Java 9+)
-Xlog:safepoint:gc-safepoint.log:time
-Xlog:gc*:gc.log:time

# Solutions
-XX:MaxGCPauseMillis=100  # Réduction target
-XX:NewRatio=1           # Plus de young gen
```

### G1GC - Mixed GC issues
```bash
# Diagnostics
-Xlog:gc*:gc.log:time
-Xlog:gc+heap=debug:gc-heap.log:time

# Solutions
-XX:G1MixedGCCountTarget=16     # Plus de mixed GC
-XX:G1OldCSetRegionThreshold=5  # Moins de régions par cycle
```

### ZGC - High allocation rate
```bash
# Diagnostics
-Xlog:gc*:gc.log:time

# Solutions
-XX:SoftMaxHeapSize=28g  # Plus de marge
-XX:ZCollectionInterval=1 # GC plus fréquent
```

## Outils de Monitoring

### GCViewer
```bash
# Export des logs (Java 9+)
-Xlog:gc*:gc.log:time

# Pour Java 8
-Xloggc:gc.log -XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=10M

# Analyse avec GCViewer
java -jar gcviewer.jar gc.log
```

### JProfiler / VisualVM
- Monitoring en temps réel
- Analyse des tendances
- Corrélation avec les métriques applicatives

### Prometheus + Grafana
```yaml
# Métriques JVM
jvm_gc_pause_seconds
jvm_gc_memory_allocated_bytes_total
jvm_gc_memory_promoted_bytes_total
```

## Conclusion

Le choix du GC dépend fortement du contexte applicatif :

- **Parallel GC** reste le meilleur choix pour les applications orientées débit
- **G1GC** offre le meilleur compromis pour la majorité des applications
- **ZGC** est indispensable pour les applications critiques en latence

La migration doit toujours être testée en profondeur avec les charges réelles de production avant déploiement.

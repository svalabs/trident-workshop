# NetApp und Trident

## Storage in Kubernetes

[Persistente Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) sind, wie der Name schon sagt, persistente Speichervolumina innerhalb Ihres Clusters, die unabhängig von Ihren Pods existieren. Sie werden entweder manuell von einem Administrator oder wie in diesem Beispiel unter Verwendung einer [Speicherklasse](https://kubernetes.io/docs/concepts/storage/storage-classes/) erstellt. Letzteres ist eine Möglichkeit für Administratoren, den auf dem Cluster eingesetzten Pods verschiedene Speicherklassen anzubieten.

Ein Persistent Volume Claim hingegen ist eine Anfrage auf einen Teil des verfügbaren Speichers, ähnlich wie ein Pod eine Anfrage auf einen Teil der verfügbaren Node-Ressourcen ist. Ein Benutzer kann Eigenschaften angeben, die der zugrundeliegende Speicher haben soll, wie die erforderliche Größe, Geschwindigkeit, Speicherort und den Zugriffsmodus. Kubernetes kümmert sich dann um die Zuweisung des Speichers zu dem Claim, indem er sich die verfügbaren Speicherklassen ansieht.

### Hands-on

Um die verfügbaren Speicherklassen anzuzeigen, geben Sie den folgenden kubectl-Befehl ein:

`kubectl get sc`

Sie sollten eine Liste erhalten, die die folgenden Elemente enthält:

```
NAME                        PROVISIONER             AGE
sf-gold                     csi.trident.netapp.io   267d
sf-silver                   csi.trident.netapp.io   267d
storage-class-nas           csi.trident.netapp.io   344d
storage-class-ssd           csi.trident.netapp.io   310d
storage-class-storagepool   csi.trident.netapp.io   344d
```

Um die Details einer Speicherklasse anzuzeigen, können Sie den folgenden kubectl-Befehl eingeben:

`kubectl describe sc storage-class-ssd`

Sie erhalten eine Reihe von Eigenschaften wie diese:

```
Name:                  storage-class-ssd
IsDefaultClass:        No
Annotations:           <none>
Provisioner:           csi.trident.netapp.io
Parameters:            backendType=ontap-nas,media=ssd
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>
```

Sie können sehen, dass Speicher, der von dieser Speicherklasse bereitgestellt wird, per Dreizack auf ein SSD zugewiesen wird.

## Erstellen eines persistenten Persistent-Volume-Claims

Lassen Sie uns jetzt versuchen, etwas Speicherplatz zuzuweisen. Der beste Weg, dies zu tun, ist die Verwendung von `kubectl apply -f` auf eine _.yaml_ Datei. Die Datei sollte das folgende Format haben:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: persistent-volume-claim-ssd
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: storage-class-ssd
``` 

- **storageClassName** definiert die Speicherklasse, die Sie verwenden möchten. Wenn sie weggelassen wird, wird stattdessen die Standard-Speicherklasse verwendet.
- **storage** definiert die Menge an Speicherplatz, die wir benötigen. In diesem Fall fordern wir ein Gibibyte Speicherplatz an. Die Angabe von 1G würde stattdessen ein Gigabyte ergeben.

### Hands-on

Sobald Sie auf unserem Demo-Rechner eingeloggt sind, können Sie den Befehl `kubectl apply -f examples/pvc.yaml` ausführen, um ein Persistent-Volume-Claim wie oben angegeben zu erstellen.

Um den neu erstellten Volume-Claim anzusehen, müssen Sie nur `kubectl get pvc` eingeben.

Es wird Ihnen eine Liste aller Volume-Claims im Standard-Namespace angezeigt:

```
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
persistent-volume-claim-ssd   Bound    pvc-[895b6bf6-d83a-4820-99cc-a01344f4d0a2]   1Gi        RWO            storage-class-ssd   9s
```

Der Zugriffsmodus RWO bedeutet, dass der persistente Volume-Claim wie angefordert ReadWriteOnce lautet, was einer von drei möglichen Zugriffsmodi ist:

- **ReadWriteOnce (RWO)** - das Volume kann von einem einzigen Knoten als Schreib-Lese-Zugriff eingelegt werden.
- **ReadOnlyMany (ROX)** - das Volume kann von vielen Knoten schreibgeschützt gemountet werden.
- **ReadWriteMany (RWX)** - das Volume kann von vielen Knoten als Lese-/Schreibzugriff gemountet werden.


## Erstellen eines Snapshots

Um einen Snapshot eines vorhandenen Volumes zu erstellen, benötigen Sie zunächst eine VolumeSnapshotClass. 
Ein VolumeSnapshotClass-Objekt ist einem StorageClass-Objekt ähnlich. Genauso wie StorageClass eine abstrakte Definition von Speicher für die Bereitstellung eines Volumes bietet, bietet VolumeSnapshotClass eine abstrakte 
Definition des Speichers für die Bereitstellung eines Snapshots.

Die Erstellung einer VolumeSnapshotClass für Trident Volumes kann mit der folgenden YAML Datei erstellt werden:

```yaml
apiVersion: snapshot.storage.k8s.io/v1alpha1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
snapshotter: csi.trident.netapp.io
```

Der Parameter **snapshotter** in dieser YAML-Datei hat den Wert `csi.trident.netapp.io`, der Kubernetes mitteilt, dass CSI Trident für die Erstellung des Snapshots für dieses Objekt zuständig ist.

Um dies auf Ihrer Demo-Maschine anzuwenden, müssen Sie lediglich `kubectl apply -f examples/snap-sc.yaml` eingeben. Sie können nun die neu erstellte VolumeSnapshotClass überprüfen, indem Sie `kubectl get volumesnapshotclass` eingeben.

```
NAME            AGE
csi-snapclass   23s
```

Jetzt, da wir einen Provider für Volume Snapshots haben, können wir dazu übergehen, tatsächlich ein Snapshot zu erstellen. 

 Dazu erstellen wir ein Objekt der VolumeSnapshot-Klasse:

```yaml
apiVersion: snapshot.storage.k8s.io/v1alpha1
kind: VolumeSnapshot
metadata:
  name: pvc1-snap
spec:
  snapshotClassName: csi-snapclass
  source:
    name: persistent-volume-claim-ssd
    kind: PersistentVolumeClaim
```

Diese spezielle VolumeSnapshot-Anforderungsdefinition nutzt die `csi-snapclass` VolumeSnapshotClass, um die Erstellung eines VolumeSnapshot-Objektnamens `pvc1-snap` gegen das PVC namens `persistent-volume-claim-ssd`, welches wir zuvor erstellt haben.

Sie können es auf Ihrem Demo-Rechner über `kubectl apply -f examples/snap.yaml` anwenden. Um alle verfügbaren Snapshots aufzulisten, verwenden Sie einfach folgenden Befehl: `kubectl get volumesnapshots`:

```
NAME        AGE
pvc1-snap   11s
```

## Wiederherstellen eines PV aus einem Snapshot

Um zu demonstrieren, wie der Snapshot wiederhergestellt wird, löschen wir nun das PersistentVolumeClaim, indem wir `kubectl delete -f examples/pvc.yaml` ausführen. Wenn Sie nun die Liste der pvc's überprüfen, werden Sie feststellen, daß es im default Namespace keine pvc's mehr gibt. 

Der Snapshot wird jedoch weiterhin sowohl in Kubernetes als auch in trident vorhanden sein. Sie können ihn verwenden:
- `kubectl get volumesnapshots` verwenden, um den Snapshot in Kubernetes anzuzeigen.
  ```
  NAME        AGE
  pvc1-snap   23s
  ```
- `tridentctl get snapshot -n trident`, um den Snapshot in Trident anzuzeigen:
  ```
  +-----------------------------------------------+------------------------------------------+
  |                     NAME                      |                  VOLUME                  |
  +-----------------------------------------------+------------------------------------------+
  | snapshot-17c50984-bba8-4b72-92a3-b2fbce71e515 | pvc-c91bd2bd-aa7c-45ac-998c-4afe25d8081e |
  +-----------------------------------------------+------------------------------------------+```

Um das persistente Volumen wiederherzustellen, erstellen wir jetzt einfach einen neuen pvc:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-from-snap
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: storage-class-ssd
  resources:
    requests:
      storage: 1Gi
  dataSource:
    name: pvc1-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

Diese YAML-Datei definiert eine Aufforderung, aus dem VolumeSnapshot-Objekt namens `pvc1-snap` ein PVC mit dem Namen `pvc-from-snap` zu erzeugen. In der YAML-Datei wird außerdem eine Volumenkapazität von 1 Gibibyte angegeben, was der ursprünglichen Kapazität des NFS-Quellvolumens von 1 Gi entspricht. Der Name des Quellvolumes, mit dem der Snapshot verknüpft ist, wird in dieser Datei nicht aufgeführt, da diese Information aus dem `pvc1-snap` VolumeSnapshot-Objekt und aus den Daten des von Trident verwalteten Objekts abgeleitet wird.

Um dies auf Ihre Demo-Maschine anzuwenden, führen Sie bitte `kubectl apply -f examples/clone.yaml` aus. Sie haben nun erfolgreich den pvc wiederhergestellt.

Um alles zu bereinigen, geben Sie bitte die folgenden Befehle ein:

```bash
kubectl delete -f examples/pvc.yaml 
kubectl delete -f examples/snap.yaml 
kubectl delete -f examples/clone.yaml 
kubectl delete -f examples/snap-sc.yaml
```

Wenn Sie sowohl `tridentctl get snapshot -n trident` als auch `tridentctl get volume -n trident` vergleichen, sehen Sie nach 30 bis 60 Sekunden, dass der Snapshot und das Volumen erfolgreich entfernt wurden.


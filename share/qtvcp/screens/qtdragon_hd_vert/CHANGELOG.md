# Changelog — qtdragon_hd_vert

---

## FIX virtual keyboard close

### Sorun
`chk_use_virtual` checkbox'ı aktifken bir `QLineEdit`'e tıklandığında sanal klavye açılıyordu, ancak:
- Klavye dışındaki **buton veya odaklanabilir alanlara** tıklandığında klavye kapanmıyordu
- Klavye dışındaki **boş alanlara** tıklandığında (odak değişmediği için) klavye kapanmıyordu

### Kök Neden

`processed_focus_event__` yalnızca `FocusIn` event'lerini handle ediyordu. `FocusOut` durumu hiç ele alınmıyordu. Boş alanlara tıklandığında ise odak değişmediğinden `FocusOut` zaten tetiklenmiyordu.

### Yapılan Değişiklikler

#### 1. `processed_focus_event__` — FocusOut desteği eklendi

```python
def processed_focus_event__(self, receiver, event):
    if not self.w.chk_use_virtual.isChecked() or STATUS.is_auto_mode(): return
    if event.type() == QtCore.QEvent.FocusIn:
        if isinstance(receiver, QtWidgets.QLineEdit):
            if not receiver.isReadOnly():
                self.w.stackedWidget_dro.setCurrentIndex(1)
        elif isinstance(receiver, QtWidgets.QTableView):
            self.w.stackedWidget_dro.setCurrentIndex(1)
        elif isinstance(receiver, QtWidgets.QCommonStyle):
            return
    elif event.type() == QtCore.QEvent.FocusOut:
        if isinstance(receiver, (QtWidgets.QLineEdit, QtWidgets.QTableView)):
            QtCore.QTimer.singleShot(0, self._check_hide_keyboard)
```

`qt_makegui.py`'daki `eventFilter`, `FocusIn` ve `FocusOut` event'lerini aynı `processed_focus_event__` fonksiyonuna yönlendiriyor. `event.type()` ile ayrım yaparak `FocusOut` durumunda `_check_hide_keyboard` zamanlayıcısı kuyruğa alındı.

`QTimer.singleShot(0, ...)` kullanılmasının nedeni: `FocusOut` tetiklendiği anda Qt henüz yeni widget'a odaklanmamıştır. 0 ms'lik timer, Qt'nin event loop'unu bir tur çalıştırıp `focusWidget()` güncellenene kadar bekler. Bu sayede iki `QLineEdit` arasında Tab ile geçişte klavye gereksiz yere kapanıp açılmaz.

#### 2. `_check_hide_keyboard` — Tab / programatik geçiş için klavyeyi kapat

```python
def _check_hide_keyboard(self):
    if not self.w.chk_use_virtual.isChecked(): return
    focused = QtWidgets.QApplication.focusWidget()
    if not isinstance(focused, (QtWidgets.QLineEdit, QtWidgets.QTableView)):
        self.w.stackedWidget_dro.setCurrentIndex(0)
```

Timer tetiklendiğinde yeni odak widget'ı kontrol edilir. Eğer `QLineEdit` veya `QTableView` değilse (yani bir sonraki odak hedefi metin girişi değilse) klavye kapatılır. `focused` değerinin `None` olması da klavyeyi kapatır — bu beklenen davranıştır.

Bu mekanizma **Tab tuşu** ve **programatik `setFocus()` çağrıları** ile tetiklenen geçişleri kapsar.

#### 3. `_KeyboardMouseFilter` — Boş alan tıklamalarını yakala

```python
class _KeyboardMouseFilter(QtCore.QObject):
    def __init__(self, handler):
        super().__init__()
        self._h = handler

    def eventFilter(self, receiver, event):
        if event.type() == QtCore.QEvent.MouseButtonPress:
            if not isinstance(event, QtGui.QMouseEvent):
                return super().eventFilter(receiver, event)
            w = self._h.w
            if w.stackedWidget_dro.currentIndex() == 1:
                is_text_input = isinstance(receiver, (QtWidgets.QLineEdit, QtWidgets.QTableView))
                if not is_text_input:
                    dro = w.stackedWidget_dro
                    kb_rect = QtCore.QRect(dro.mapToGlobal(QtCore.QPoint(0, 0)), dro.size())
                    if not kb_rect.contains(event.globalPos()):
                        w.stackedWidget_dro.setCurrentIndex(0)
        return super().eventFilter(receiver, event)
```

`QApplication.instance()` üzerine kurulan global event filter, uygulamadaki tüm `MouseButtonPress` event'lerini yakalar. Odak değiştirmeyen boş alanlara tıklama da bu sayede yakalanır.

**Koordinat tabanlı kontrol tercih edilmesinin nedeni:** İlk tasarımda klavye widget'ının parent zinciri (`_inside_keyboard`) traversal ile kontrol ediliyordu. Ancak `virtual_keyboard.ui`'daki `gridWidget`'ın `native="true"` özelliği nedeniyle bu yöntem güvenilir çalışmadı — native widget'lar ayrı OS penceresi oluşturduğundan Qt'nin widget parent zinciri kesilebilir.

Bunun yerine `stackedWidget_dro`'nun ekran koordinatları `mapToGlobal()` ile hesaplanır ve tıklama noktasının bu rect içinde olup olmadığı `QRect.contains()` ile kontrol edilir. `isinstance(event, QtGui.QMouseEvent)` type guard ile beklenmedik event tiplerinden korunulur.

Filter, `initialized__` içinde kurulur ve handler örneğinde tutulur:

```python
self._kb_mouse_filter = _KeyboardMouseFilter(self)
QtWidgets.QApplication.instance().installEventFilter(self._kb_mouse_filter)
```

### Kapsam Tablosu

| Senaryo | Mekanizma |
|---|---|
| Text input'a tıkla → klavye aç | `processed_focus_event__` FocusIn |
| Focuslanabilir widget'a tıkla → kapat | `_check_hide_keyboard` (FocusOut) |
| Tab ile text input dışına çık → kapat | `_check_hide_keyboard` (FocusOut) |
| Programatik `setFocus()` → kapat | `_check_hide_keyboard` (FocusOut) |
| Boş alana tıkla → kapat | `_KeyboardMouseFilter` (koordinat) |
| Klavye butonuna tıkla → açık kal | `_KeyboardMouseFilter` (kb_rect içi) |
| İki text input arası Tab → açık kal | `_check_hide_keyboard` (focusWidget kontrolü) |

---

## FIX clearFocus for text inputs

### Sorun

Klavye açıkken boş bir alana tıklandığında `_KeyboardMouseFilter` klavyeyi kapatıyordu, ancak text input **fokuslu kalmaya devam ediyordu**. Boş alanlar odaklanamayan widget'lar olduğundan Qt'de fokus değişmez; dolayısıyla `FocusOut` tetiklenmez ve text input fokusunu korur.

Bu durumda:
1. Text input'a tıkla → klavye açılır
2. Boş alana tıkla → klavye kapanır, **text input fokuslu kalır**
3. Aynı text input'a tekrar tıkla → `FocusIn` tetiklenmez (fokus zaten orada), klavye açılmaz

### Kök Neden

Qt'de `FocusIn` yalnızca fokus **değiştiğinde** tetiklenir. Text input zaten fokuslu olduğu için bir sonraki tıklamada `FocusIn` event'i üretilmez ve `processed_focus_event__` hiç çağrılmaz.

### Yapılan Değişiklik

`_KeyboardMouseFilter.eventFilter` içinde, klavye kapatılırken mevcut odaklı widget'ın fokusunun da temizlenmesi sağlandı:

```python
if not kb_rect.contains(event.globalPos()):
    w.stackedWidget_dro.setCurrentIndex(0)
    focused = QtWidgets.QApplication.focusWidget()
    if focused is not None:
        focused.clearFocus()
```

### Çalışma Şekli

Event filter, mouse press event receiver'a teslim edilmeden **önce** çalışır. Bu sırada:

1. `stackedWidget_dro.setCurrentIndex(0)` → klavye kapatılır
2. `QApplication.focusWidget()` → o an fokuslu widget alınır (text input)
3. `focused.clearFocus()` → text input'un fokus durumu temizlenir
4. Mouse press event boş alana teslim edilir → boş alan odaklanamadığından yeni bir fokus oluşmaz
5. Sonuç: hiçbir widget fokuslu değil

Bir sonraki tıklamada text input yeniden tıklandığında Qt `FocusIn` event'i üretir → `processed_focus_event__` tetiklenir → klavye açılır.

### Güvenlik

`focused is not None` kontrolü, uygulama genelinde hiçbir widget fokuslu olmadığı durumda `clearFocus()` çağrısının atlanmasını sağlar.

### Güncel Kapsam Tablosu

| Senaryo | Mekanizma |
|---|---|
| Text input'a tıkla → klavye aç | `processed_focus_event__` FocusIn |
| Focuslanabilir widget'a tıkla → kapat | `_check_hide_keyboard` (FocusOut) |
| Tab ile text input dışına çık → kapat | `_check_hide_keyboard` (FocusOut) |
| Programatik `setFocus()` → kapat | `_check_hide_keyboard` (FocusOut) |
| Boş alana tıkla → kapat + fokus temizle | `_KeyboardMouseFilter` (koordinat + clearFocus) |
| Klavye butonuna tıkla → açık kal | `_KeyboardMouseFilter` (kb_rect içi) |
| İki text input arası Tab → açık kal | `_check_hide_keyboard` (focusWidget kontrolü) |
| Boş alan sonrası text input'a tekrar tıkla → klavye açılır | clearFocus → FocusIn tetiklenir |

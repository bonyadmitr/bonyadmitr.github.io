# Тап жест для скрытия клавиатуры в iOS (Swift 3)

В данной статье разберем, как скрывать клавиатуру по нажатию на вьюху от самых основ до реализации в одну строчку или совсем без кода.

![kdpv](https://raw.githubusercontent.com/bonyadmitr/KeyboardHideManager/develop/Resources/kdpv.png)

<habracut/>

## Основы

Достаточно часто встречается следующий кейс: по нажатию на задний фон скрывать клавиатуру.

Базовое решение - имеем ссылку на `UITextField`, создаем `UITapGestureRecognizer` с методом, который снимает выделение с текстового поля и выглядит оно так:

> В данной статье используется Swift 3, но можно реализовать и на других версиях и на Objective-C

```swift
class ViewController: UIViewController {
    
    @IBOutlet weak var textField: UITextField!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let tapGesture = UITapGestureRecognizer(target: self, action: #selector(self.tapGesture))
        view.addGestureRecognizer(tapGesture)
    }
    
    func tapGesture() {
        textField.resignFirstResponder()
    }
}
```

Проблемы данного кода:

- `viewDidLoad` грязный и нечитабельный
- много кода в контроллере
- не переиспользуемый

## Делаем читабельным

Для решения первой проблемы мы можем вынести код создания и добавления жеста в отдельную функцию:

```swift
class ViewController: UIViewController {
    
    @IBOutlet weak var textField: UITextField!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        addTapGestureToHideKeyboard()
    }
    
    func addTapGestureToHideKeyboard() {
        let tapGesture = UITapGestureRecognizer(target: self, action: #selector(self.tapGesture))
        view.addGestureRecognizer(tapGesture)
    }
    
    func tapGesture() {
        textField.resignFirstResponder()
    }
}
```

Кода стало еще больше, но он стал чище, логичней и приятней глазу.

## Уменьшение кода

Для решения второй проблемы у `UIView` есть метод:

```swift
func endEditing(_ force: Bool) -> Bool
```

Он как раз отвечает за снятие выделения с самой вьюхи или ее subview. Благодаря нему мы можем сильно упростить наш код:

```swift
class ViewController: UIViewController {
    
    override func viewDidLoad() {
        super.viewDidLoad()
        addTapGestureToHideKeyboard()
    }
    
    func addTapGestureToHideKeyboard() {
        let tapGesture = UITapGestureRecognizer(target: view, action: #selector(view.endEditing))
        view.addGestureRecognizer(tapGesture)
    }
}
```

> Если делаете по шагам, то не забудьте удалить textField свойство из IB.
> Также поменяйте `target` с `self` на `view`.

Код стал радовать глаз! Но копировать это в каждый контроллер все еще придется.

## Решение копирования

Для переиспользуемости вынесем наш метод добавления в `extension` контроллера:

```swift
extension UIViewController {
    func addTapGestureToHideKeyboard() {
        let tapGesture = UITapGestureRecognizer(target: view, action: #selector(view.endEditing))
        view.addGestureRecognizer(tapGesture)
    }
}
```

И код нашего контроллера будет выглядеть следующим образом:

```swift
class ViewController: UIViewController {
    
    override func viewDidLoad() {
        super.viewDidLoad()
        addTapGestureToHideKeyboard()
    }
}
```

Чисто, одна строчка кода, и она переиспользуема! Идеально!

## Несколько вьюх

Решение выше - очень хорошее, но в нем кроется один минус: мы не можем добавить жест на конкретную вьюху.

Для решения данного кейса воспользуемся расширением `UIView`:

```swift
extension UIView {
    func addTapGestureToHideKeyboard() {
        let tapGesture = UITapGestureRecognizer(target: self, action: #selector(endEditing))
        addGestureRecognizer(tapGesture)
    }
}
```

и соответственно код контроллера будет выглядеть так:

```swift
class ViewController: UIViewController {
    
    override func viewDidLoad() {
        super.viewDidLoad()
        view.addTapGestureToHideKeyboard()
    }
}
```

Тут возникает другая проблема: данное расширение решает проблему только для вьюхи контроллера. Если мы добавить someView на view и на нее повесим жест, то не сработает. Это все из-за того, что метод `endEditing` работает только для вьюхи, которая содержит активную вьюху или сама таковой является, а нашего текстового поля скорее всего не будет в нем. Решим данную проблему.

Т.к. view контроллера точно будет содержать активную вьюху, и наша добавленная вьюха всегда будет в ее иерархии, то мы можем дотянуться до view контроллера через `superview ` и у нее вызвать `endEditing`.

Получаем view контроллера через расширение UIView:

```swift
var topSuperview: UIView? {
    var view = superview
    while view?.superview != nil {
        view = view!.superview
    }
    return view
}
```

Скажу сразу, изменив селектор на:

```swift
#selector(topSuperview?.endEditing)
```
работать все еще не будет. Нам необходимо добавить метод, который будет вызывать конструкцию выше:

```swift
func dismissKeyboard() {
    topSuperview?.endEditing(true)
}
```

Вот теперь заменяем селектор на:

```swift
#selector(dismissKeyboard)
```

Итак, расширение для UIView будет выглядеть следующим образом:

```swift
extension UIView {
    
    func addTapGestureToHideKeyboard() {
        let tapGesture = UITapGestureRecognizer(target: self, action: #selector(dismissKeyboard))
        addGestureRecognizer(tapGesture)
    }
    
    var topSuperview: UIView? {
        var view = superview
        while view?.superview != nil {
            view = view!.superview
        }
        return view
    }
    
    func dismissKeyboard() {
        topSuperview?.endEditing(true)
    }
}
```

Теперь, используя `addTapGestureToHideKeyboard()` для любой вьюхи мы будем скрывать клавиатуру.

## KeyboardHideManager

![Icon](https://raw.githubusercontent.com/bonyadmitr/KeyboardHideManager/develop/Resources/keyboard_icon.png)

Решением выше я пользовался долгое время, но потом стал замечать, что даже одна строка загрязняет функцию установки вьюх. Так же, (редко, но все же бывает) не очень красиво выглядит, когда это единственный метод во `viewDidLoad`:

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    addTapGestureToHideKeyboard()
}
```

Вместе с пробелом это занимает 5 строк, что сильно сказывается на чистоте контроллера!

У меня появилась идея сделать все это без кода, чтобы не было в контроллере и одной лишней строчки.

Я создал класс, который можно добавить в IB с помощью **Object** 

![object](https://raw.githubusercontent.com/bonyadmitr/KeyboardHideManager/develop/Resources/object.png)

и с помощью `@IBOutlet` привязать нужные нам вьюхи:

![preview](https://raw.githubusercontent.com/bonyadmitr/KeyboardHideManager/develop/Resources/preview.png)

Реализация данного класса наипростейшая - добавляем жест на каждую вьюху с помощью выше созданных функций:

```swift
final public class KeyboardHideManager: NSObject {
    
    @IBOutlet internal var targets: [UIView]! {
        didSet {
            for target in targets {
                addGesture(to: target)
            }
        }
    }
    
    internal func addGesture(to target: UIView) {
        let gesture = UITapGestureRecognizer(target: self, action: #selector(dismissKeyboard))
        target.addGestureRecognizer(gesture)
    }
    
    @objc internal func dismissKeyboard() {
        targets.first?.topSuperview?.endEditing(true)
    }
}

extension UIView {
    internal var topSuperview: UIView? {
        var view = superview
        while view?.superview != nil {
            view = view!.superview
        }
        return view
    }
}
```

Чтобы воспользоваться данным классом, нужно три простых действия:

- 1) Перетащить **Object** в контроллер

![usage_1](https://raw.githubusercontent.com/bonyadmitr/KeyboardHideManager/develop/Resources/usage_1.png)

- 2) Установить **KeyboardHideManager** в качестве класса данного объекта

![usage_2](https://raw.githubusercontent.com/bonyadmitr/KeyboardHideManager/develop/Resources/usage_2.png)

- 3) Соединить нужные вьюхи со свойством **targets**

![usage_3](https://raw.githubusercontent.com/bonyadmitr/KeyboardHideManager/develop/Resources/usage_3.png)

Да, больше действий, чем написать одну строку (или несколько строк, если несколько определенных вьюх), за то этой самой строки нету в контроллере.

Кто-то может сказать, что лучше написать строку в коде, так понятнее, и будут правы, с одной стороны. Я за то, чтобы вынести из контроллера, все что можно и тем самым его облегчить.

Данный класс можете подключить через CocoaPods или просто скопировать в проект.

[Ссылка](https://github.com/bonyadmitr/KeyboardHideManager) на исходный код **KeyboardHideManager** с полным ReadMe о самой библиотеке и ее подключении.

## Итог

Разобрали реализацию популярного кейса, рассмотрели несколько его решений, в одну строку и без кода вообще. Используйте тот способ, который больше нравится.
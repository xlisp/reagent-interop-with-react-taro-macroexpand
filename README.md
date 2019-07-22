# reagent-interop-with-react-taro: 网页和安卓和iOS的React Native和微信小程序taro的同时编译输出, taro只不过是类react语法编译输出为wxss编译输出相同的js,React只不过是jsx编译输出为js

* 思想：先跑起来安卓的Java代码，然后用互操作来逐一替换为Clojure

#### [Motivation](https://github.com/binaryage/cljs-oops)

> I don't always do Javascript interops, but when I do, I call them by names.
>
> -- <cite>Darwin (with sunglasses on)</cite>

ClojureScript developers should quickly learn how to inter-operate with native Javascript objects via [the dot operator][1].
This was modelled to closely follow [Clojure's Java interop][4] story.

For example, the ClojureScript form `(.-nativeProp obj)` will compile to `obj.nativeProp` in Javascript.

It works pretty well [during development][3] but there is a catch! When you naively write code like that, it might
not survive [advanced optimizations][2]. Closure Compiler needs some information about which property names are safe to rename
and which cannot be renamed because they might be referenced externally or dynamically via strings.

Someone at Google had a quick and bad idea. We could provide a separate file which would describe this information.
Let's call it an "externs file"!

* 一个最快跑起来的完整的CURD的Rails + React + Material UI 的 CURD的最简例子
* 如何兴趣转移? 学习转移之前最重要的: 不可变数据结构(响应式编程)this.state, React组件的函数复用思想

* [Reagent Interop With React](http://reagent-project.github.io/docs/master/InteropWithReact.html "Permalink to Interop with React")
* [Taro](https://github.com/NervJS/taro)
* [Reagent demo](https://github.com/reagent-project/reagent/tree/master/demo), [Reagent doc](https://github.com/reagent-project/reagent/tree/master/doc), [Reagent examples](https://github.com/reagent-project/reagent/tree/master/examples)

# 领域化macroexpand + 万能的数据结构S表达式的List, 通过代码语义搜索reagent来翻译React代码(代码语义搜索是最好的文档): 改变语法本身来快速入侵新的其它领域

```clojure
(ns reagent.ratom
  (:refer-clojure :exclude [run!])
  (:require [reagent.debug :as d]))

(defmacro reaction [& body]
  `(reagent.ratom/make-reaction
    (fn [] ~@body)))

(defmacro run!
  "Runs body immediately, and runs again whenever atoms deferenced in the body change. Body should side effect."
  [& body]
  `(let [co# (reagent.ratom/make-reaction (fn [] ~@body)
                                         :auto-run true)]
     (deref co#)
     co#))

; taken from cljs.core
; https://github.com/binaryage/cljs-oops/issues/14
(defmacro unchecked-aget
  ([array idx]
   (list 'js* "(~{}[~{}])" array idx))
  ([array idx & idxs]
   (let [astr (apply str (repeat (count idxs) "[~{}]"))]
     `(~'js* ~(str "(~{}[~{}]" astr ")") ~array ~idx ~@idxs))))

; taken from cljs.core
; https://github.com/binaryage/cljs-oops/issues/14
(defmacro unchecked-aset
  ([array idx val]
   (list 'js* "(~{}[~{}] = ~{})" array idx val))
  ([array idx idx2 & idxv]
   (let [n (dec (count idxv))
         astr (apply str (repeat n "[~{}]"))]
     `(~'js* ~(str "(~{}[~{}][~{}]" astr " = ~{})") ~array ~idx ~idx2 ~@idxv))))

(defmacro with-let [bindings & body]
  (assert (vector? bindings)
          (str "with-let bindings must be a vector, not "
               (pr-str bindings)))
  (let [v (gensym "with-let")
        k (keyword v)
        init (gensym "init")
        bs (into [init `(zero? (alength ~v))]
                 (map-indexed (fn [i x]
                                (if (even? i)
                                  x
                                  (let [j (quot i 2)]
                                    `(if ~init
                                       (unchecked-aset ~v ~j ~x)
                                       (unchecked-aget ~v ~j)))))
                              bindings))
        [forms destroy] (let [fin (last body)]
                          (if (and (list? fin)
                                   (= 'finally (first fin)))
                            [(butlast body) `(fn [] ~@(rest fin))]
                            [body nil]))
        add-destroy (when destroy
                      `(let [destroy# ~destroy]
                         (if (reagent.ratom/reactive?)
                           (when (nil? (.-destroy ~v))
                             (set! (.-destroy ~v) destroy#))
                           (destroy#))))
        asserting (if *assert* true false)]
    `(let [~v (reagent.ratom/with-let-values ~k)]
       (when ~asserting
         (when-some [^clj c# reagent.ratom/*ratom-context*]
           (when (== (.-generation ~v) (.-ratomGeneration c#))
             (d/error "Warning: The same with-let is being used more "
                      "than once in the same reactive context."))
           (set! (.-generation ~v) (.-ratomGeneration c#))))
       (let ~bs
         (let [res# (do ~@forms)]
           ~add-destroy
           res#)))))
           
;; ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
(ns reagent.core
  (:require [reagent.ratom :as ra]))
(defmacro with-let
  "Bind variables as with let, except that when used in a component
  the bindings are only evaluated once. Also takes an optional finally
  clause at the end, that is executed when the component is
  destroyed."
  [bindings & body]
  `(ra/with-let ~bindings ~@body))
  
;; ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;             
(ns reagent.core
  (:require-macros [reagent.core]) ;; 一个with-let宏
  (:refer-clojure :exclude [partial atom flush]) ;;避免和clojure自身的clojure冲突
  (:require [react :as react]
            [reagent.impl.template :as tmpl]
            [reagent.impl.component :as comp]
            [reagent.impl.util :as util]
            [reagent.impl.batching :as batch]
            [reagent.ratom :as ratom]            
            ;; 下面是宏最多的地方
            [reagent.debug :as deb :refer-macros [dbg prn
                                                  assert-some assert-component
                                                  assert-js-object assert-new-state
                                                  assert-callable]]
            [reagent.dom :as dom]))
            
(defn create-element
  "Create a native React element, by calling React.createElement directly.

  That means the second argument must be a javascript object (or nil), and
  that any Reagent hiccup forms must be processed with as-element. For example
  like this:

  (r/create-element \"div\" #js{:className \"foo\"}
    \"Hi \" (r/as-element [:strong \"world!\"])

  which is equivalent to

  [:div.foo \"Hi\" [:strong \"world!\"]]   "
  ([type]
   (create-element type nil))
  ([type props]
   (assert-js-object props)
   (react/createElement type props))
  ([type props child]
   (assert-js-object props)
   (react/createElement type props child))
  ([type props child & children]
   (assert-js-object props)
   (apply react/createElement type props child children)))

(defn as-element
  "Turns a vector of Hiccup syntax into a React element. Returns form
  unchanged if it is not a vector."
  [form]
  (tmpl/as-element form))

(defn adapt-react-class
  "Returns an adapter for a native React class, that may be used
  just like a Reagent component function or class in Hiccup forms."
  [c]
  (assert-some c "Component")
  (tmpl/adapt-react-class c))

(defn reactify-component
  "Returns an adapter for a Reagent component, that may be used from
  React, for example in JSX. A single argument, props, is passed to
  the component, converted to a map."
  [c]
  (assert-some c "Component")
  (comp/reactify-component c))

(defn render
  "Render a Reagent component into the DOM. The first argument may be
  either a vector (using Reagent's Hiccup syntax), or a React element.
  The second argument should be a DOM node.

  Optionally takes a callback that is called when the component is in place.

  Returns the mounted component instance."
  ([comp container]
   (dom/render comp container))
  ([comp container callback]
   (dom/render comp container callback)))

(defn state
  "Returns the state of a component, as set with replace-state or set-state.
  Equivalent to `(deref (r/state-atom this))`"
  [this]
  (assert-component this)
  (deref (state-atom this)))

(defn set-state
  "Merge component state with new-state.
  Equivalent to `(swap! (state-atom this) merge new-state)`"
  [this new-state]
  (assert-component this)
  (assert-new-state new-state)
  (swap! (state-atom this) merge new-state))

(defn props
  "Returns the props passed to a component."
  [this]
  (assert-component this)
  (comp/get-props this))

;; ;;;;;;;;;;;;;impl
(defn adapt-react-class
  [c]
  (->NativeWrapper c nil nil))
(deftype NativeWrapper [tag id className])  
;; ;;
(defn dom-node
  "Returns the root DOM node of a mounted component."
  [this]
  (react-dom/findDOMNode this))
;; ;;;;;;;;;;;; reactify-component的简介接口的背后,组件就是为了实现函数式思想的复用组合 ;;;;;;;;;;;;;;;
;; 强大的 [goog.object :as gobj] 
(defn create-class
  "Creates JS class based on provided Clojure map.

  Map keys should use `React.Component` method names (https://reactjs.org/docs/react-component.html),
  and can be provided in snake-case or camelCase.
  Constructor function is defined using key `:getInitialState`.

  React built-in static methods or properties are automatically defined as statics."
  [body]
  {:pre [(map? body)]}
  (let [body (cljsify body)
        methods (map-to-js (apply dissoc body :displayName :getInitialState
                                  :render :reagentRender
                                  built-in-static-method-names))
        static-methods (map-to-js (select-keys body built-in-static-method-names))
        display-name (:displayName body)
        construct (:getInitialState body)
        cmp (fn [props context updater]
              (this-as this
                (.call react/Component this props context updater)
                (when construct
                  (construct this))
                this))]

    (gobj/extend (.-prototype cmp) (.-prototype react/Component) methods)

    ;; These names SHOULD be mangled by Closure so we can't use goog/extend

    (when (:render body)
      (set! (.-render (.-prototype cmp)) (:render body)))

    (when (:reagentRender body)
      (set! (.-reagentRender (.-prototype cmp)) (:reagentRender body)))

    (when (:cljsLegacyRender body)
      (set! (.-cljsLegacyRender (.-prototype cmp)) (:cljsLegacyRender body)))

    (gobj/extend cmp react/Component static-methods)

    (when display-name
      (set! (.-displayName cmp) display-name)
      (set! (.-cljs$lang$ctorStr cmp) display-name)
      (set! (.-cljs$lang$ctorPrWriter cmp)
            (fn [this writer opt]
              (cljs.core/-write writer display-name))))

    (set! (.-cljs$lang$type cmp) true)
    (set! (.. cmp -prototype -constructor) cmp)

    cmp))
    
(defn fn-to-class [f]
  (assert-callable f)
  (warn-unless (not (and (react-class? f)
                         (not (reagent-class? f))))
               "Using native React classes directly in Hiccup forms "
               "is not supported. Use create-element or "
               "adapt-react-class instead: " (or (util/fun-name f)
                                                 f)
               (comp-name))
  (if (reagent-class? f)
    (cache-react-class f f)
    (let [spec (meta f)
          withrender (assoc spec :reagent-render f)
          res (create-class withrender)]
      (cache-react-class f res))))

(defn as-class [tag]
  (if-some [cached-class (cached-react-class tag)]
    cached-class
    (fn-to-class tag)))

(defn reactify-component [comp]
  (if (react-class? comp)
    comp
    (as-class comp)))
```

* goog.object

```clojure
;; [goog.object :as gobj]
(def custom-theme
  (createMuiTheme
   #js {:palette #js {:primary #js {:main (gobj/get (.-red mui-colors) 100)}}}))
   
```

# Interop with React

A little understanding of what Reagent is doing really helps when trying to use React libraries and reagent together.

## Creating React Elements directly

The `reagent.core/create-element` function simply calls React's `createElement` function (and therefore, it expects either a string representing an HTML element or a React Component).

As an example, here are four ways to create the same element:
    
```clojure
(defn integration []
 [:div
  [:div.foo "Hello " [:strong "world"]]

  (r/create-element "div"
                    #js{:className "foo"}
                    "Hello "
                    (r/create-element "strong"
                                       #js{}
                                       "world"))

  (r/create-element "div"
                    #js{:className "foo"}
                    "Hello "
                    (r/as-element [:strong "world"]))
;;(r/as-element [:strong "world"])
;; => #js {"$$typeof" "Symbol(react.element)", :type "strong", :key nil, :ref nil, :props #js {:children "world"}, :_owner nil, :_store #js {}}

;; create-element 
;;#js {"$$typeof" "Symbol(react.element)", :type "div", :key nil, :ref nil, :props #js {:className "foo",
;;                                                                                      :children #js ["Hello "
;;           #js {"$$typeof" "Symbol(react.element)", :type "strong", :key nil, :ref nil, :props #js {:children "world"}, :_owner nil, :_store #js {}}]}, :_owner nil, :_store #js {}}

  [:div.foo "Hello " (r/create-element "strong"
                                       #js{}
                                       "world")]])

(defn mount-root []
 (reagent/render [integration]
   (.getElementById js/document "app")))
```

This works because `reagent/render` itself expects (1) a React element or (2) a Hiccup form. If passed an element, it just uses it. If passed a Hiccup, it creats a (cached) React component and then creates an element from that component.

## Creating React Elements from Hiccup forms

The `reagent.core/as-element` function creates a React element from a Hiccup form. In the previous section, we discussed how `reagent/render` expects either (1) a Hiccup form or (2) a React Element. If it encounters a Hiccup form, it calls `as-element` on it. When you have a React component that wraps children, you can pass Hiccup forms to it wrapped in `as-element`.

## Creating Reagent "Components" from React Components

The function `reagent/adapt-react-class` will turn a React Component into something that can be placed into the first position of a Hiccup form, as if it were a Reagent function. Take, for example the react-flip-move library and assume that it has been properly imported as a React Component called `FlipMove`. By wrapping FlipMove with `adapt-react-class`, we can use it in a Hiccup form:
    
```clojure
(defn top-articles [articles]
  [(reagent/adapt-react-class FlipMove)
   {:duration 750
    :easing "ease-out"}
   articles]
```

There is also a convenience mechanism `:>` (colon greater-than) that shortens this and avoid some parenthesis:
    
```clojure
(defn top-articles [articles]
 [:> FlipMove
  {:duration 750
   :easing "ease-out"}
  articles]
```

This is the equivalent JavaScript:
    
```js
const TopArticles = ({ articles }) => (
  
    {articles}
  
);
```

```clojure
;; src/reagent/impl/template.cljs
(defn str-coll [coll]
  (if (dev?)
    (str (prewalk (fn [x]
                    (if (fn? x)
                      (let [n (util/fun-name x)]
                        (case n "" x (symbol n)))
                      x)) coll))
    (str coll)))

(defn hiccup-err [v & msg]
  (str (apply str msg) ": " (str-coll v) "\n" (comp/comp-name)))

(defn vec-to-elem [v]
  (assert (pos? (count v)) (hiccup-err v "Hiccup form should not be empty"))
  (let [tag (nth v 0 nil)]
    (assert (valid-tag? tag) (hiccup-err v "Invalid Hiccup form"))
    (cond
      (keyword-identical? :<> tag)
      (fragment-element v)

      (hiccup-tag? tag)
      (let [n (name tag)
            pos (.indexOf n ">")]
        (case pos
          -1 (native-element (cached-parse n) v 1)
          0 (let [component (nth v 1 nil)]
              ;; Support [:> component ...]
              (assert (= ">" n) (hiccup-err v "Invalid Hiccup tag"))
              (native-element (->HiccupTag component nil nil nil) v 2))
          ;; Support extended hiccup syntax, i.e :div.bar>a.foo
          ;; Apply metadata (e.g. :key) to the outermost element.
          ;; Metadata is probably used only with sequeneces, and in that case
          ;; only the key of the outermost element matters.
          (recur (with-meta [(subs n 0 pos)
                             (assoc (with-meta v nil) 0 (subs n (inc pos)))]
                            (meta v)))))

      (instance? NativeWrapper tag)
      (native-element tag v 1)

      :else (reag-element tag v))))
```

## Creating React Components from Reagent "Components": React组件就是函数复用的思想,写一个组件可以复用组合

The `reagent/reactify-component` will take a Form-1, Form-2, or Form-3 reagent "component". For example:
    
``` clojure
;; 创建一个组件基本方法 (def component (r/reactify-component (fn [props] ..)))
;; 创建一个element基本方法 (r/create-element component #js{:name "world"})
;; 其他element创建 (apply r/create-element mui/TextField props (map r/as-element children))
(defn exported [props]
  [:div "Hi, " (:name props)])

(def react-comp (r/reactify-component exported))

(defn could-be-jsx []
  (r/create-element react-comp #js{:name "world"}))
```

Note:

* `adapt-react-class` and `reactify-component` are not perfectly symmetrical, because `reactify-component` requires that the reagent component accept everything in a single props map, including its children.

## Example: "Decorator" Higher-Order Components

Some React libraries use the decorator pattern: a React component which takes a component as an argument and returns a new component as its result. One example is the React DnD library. We will need to use both `adapt-react-class` and `reactify-component` to move back and forth between React and reagent:
    
```clojure
(def react-dnd-component
  (let [decorator (DragDropContext HTML5Backend)]
    (reagent/adapt-react-class
      (decorator (reagent/reactify-component top-level-component)))))
```

This is the equivalent JavaScript:
    
```js
import HTML5Backend from 'react-dnd-html5-backend';
import { DragDropContext } from 'react-dnd';

class TopLevelComponent {
  /* ... */
}

export default DragDropContext(HTML5Backend)(TopLevelComponent);
```

## Example: Function-as-child Components

Some React components expect a function as their only child. React AutoSizer is one such example.
    
```clojure
[(reagent/adapt-react-class AutoSizer)
 {}
 (fn [dims]
  (let [dims (js->clj dims :keywordize-keys true)]
   (reagent/as-element [my-component (:height dims)])))]
```

## Getting props and children of current component

Because you just pass arguments to reagent functions, you typically don't need to think about "props" and "children" as distinct things. But Reagent does make a distinction and it is helpful to understand this, particularly when interoperating with native elements and React libraries.

Specifically, if the first argument to your Reagent function is a map, that is assigned to `this.props` of the underlying Reagent component. All other arguments are assigned as children to `this.props.children`.

When interacting with native React components, it may be helpful to access props and children, which you can do with `reagent.core/current-component`. This function returns an object that allows you retrieve the props and children passed to the current component.

Beware that `current-component` is only valid in component functions, and must be called outside of e.g. event handlers and `for` expressions, so it's safest to always put the call at the top, as in `my-div` here:
    
    
```clojure 
(ns example
  (:require [reagent.core :as r]))

(defn my-div []
  (let [this (r/current-component)]
    (into [:div.custom (r/props this)]
          (r/children this))))

(defn call-my-div []
  [:div
    [my-div "Some text."]
    [my-div {:style {:font-weight 'bold}}
      [:p "Some other text in bold."]]])
```

## React Features

[React Features and how to use them in Reagent](http://reagent-project.github.io/docs/master/ReactFeatures.html)

## Examples

### [Material-UI](https://github.com/reagent-project/reagent/blob/master/doc/examples/material-ui.md)

* 图片列表的显示 

```clojure
(defn tile [painting]
  [ui/GridTile
   {:key (:jpg painting)}
   [:img {:src (:jpg painting)
          :on-click #(js/alert 1111)}]])

(defn grid []
  (let [paintings
        [{:jpg  "https://res.cloudinary.com/dgpqnl8ul/image/upload/v1546472422/vqutvlotsmipx7xkmxun.jpg"}
         {:jpg  "https://res.cloudinary.com/dgpqnl8ul/image/upload/v1546472443/gqwvv3nmczfqaqf3c7l1.jpg"}
         {:jpg  "https://res.cloudinary.com/dgpqnl8ul/image/upload/v1546472446/x8crdusi0cqdu3rywwv8.jpg"}
         {:jpg  "https://res.cloudinary.com/dgpqnl8ul/image/upload/v1546472448/xxaoqvs6ad2av4t2aduh.jpg"}
         {:jpg  "https://res.cloudinary.com/dgpqnl8ul/image/upload/v1546472449/k3jixe3wrjb4qcuuvo3h.jpg"}
         {:jpg  "https://res.cloudinary.com/dgpqnl8ul/image/upload/v1546472451/b8fiba0shz0wldpk0bbb.jpg"}]]
    [:div
     [ui/MuiThemeProvider
      [ui/GridList {:cellHeight 160 :cols 2}
       (for [painting paintings] ^{:key (:jpg painting)} [tile painting])]]]))

(reagent/render-component [grid]
                          (. js/document (getElementById "app")))
```

* 文本输入框

```clojure
(ns example
  (:require [reagent.core  :as reagent]
            [re-frame.core :refer [subscribe dispatch]]
            ["material-ui" :as mui]
            ["material-ui-icons" :as mui-icons]
            [reagent.impl.template :as rtpl]
            [clojure.string :as str]))

;; material ui TextField的例子: https://github.com/reagent-project/reagent/blob/master/doc/examples/material-ui.md
(def text-field-1 (reagent/adapt-react-class mui/TextField)) ;; 等效于React的jsx标签<TextField ...> 或者 等效于 [:> mui/TextField ...]
(def value (reagent/atom ""))
(def input-component-1
  (reagent/reactify-component
   (fn [props]
     [:input (-> props
                 (assoc :ref (:inputRef props))
                 (dissoc :inputRef))])))

;; 可以直接被挂载到div上面
[text-field-1
      {:value @value
       :on-change #(reset! value (.. % -target -value))
       :InputProps {:inputComponent input-component-1}}]
                 
```

* 表单提交

```clojure

(defonce text-jedi-state (reagent/atom "Wage peace."))
(defonce text-sith-state (reagent/atom "Anger is useful—unless it is used against you."))

(defonce text-state (reagent/atom @text-jedi-state))

(defonce race (reagent/atom "Jedi"))

(defonce text-area-state (reagent/atom "Shaggy giants from an arboreal world, the tall and commanding Wookiee species is an impressive sight to even the most jaded spacer.\n\nDespite their fearsome and savage countenance, Wookiees are intelligent, sophisticated, loyal and trusting. Loyalty and bravery are near-sacred tenets in Wookiee society.\n\nWhen peaceful, Wookiees are tender and gentle. Their tempers, however, are short; when angered, Wookiees can fly into a berserker rage and will not stop until the object of their distemper is sufficiently destroyed."))

(def ^:private input-component
  (reagent/reactify-component
   (fn [props]
     [:input (-> props
                 (assoc :ref (:inputRef props))
                 (dissoc :inputRef))])))

(def ^:private textarea-component
  (reagent/reactify-component
   (fn [props]
     [:textarea (-> props
                    (assoc :ref (:inputRef props))
                    (dissoc :inputRef))])))

;; To fix cursor jumping when controlled input value is changed,
;; use wrapper input element created by Reagent instead of
;; letting Material-UI to create input element directly using React.
;; Create-element + convert-props-value is the same as what adapt-react-class does.
(defn text-field [props & children]
  (let [props (-> props
                  (assoc-in [:InputProps :inputComponent] (cond
                                                            (and (:multiline props) (:rows props) (not (:maxRows props)))
                                                            textarea-component

                                                            ;; FIXME: Autosize multiline field is broken.
                                                            (:multiline props)
                                                            nil

                                                            ;; Select doesn't require cursor fix so default can be used.
                                                            (:select props)
                                                            nil

                                                            :else
                                                            input-component))
                  rtpl/convert-prop-value)]
    (apply reagent/create-element mui/TextField props (map reagent/as-element children))))


(defn demo-text-field [{:keys [classes] :as props}]
  (let [component-state (reagent/atom {:selected 0})]
    (fn []
      (let [current-select (get @component-state :selected)]
        [:div {:style {:display "flex"
                       :flexDirection "column"
                       :position "relative"
                       :margin 50
                       :alignItems "left"
                       }}
         [:> mui/Grid
          {:container true
           :direction "column"}
          [:h2 "发布活动"]
          [:div {:style {:margin-bottom 20}}
           [text-field
            {:value @text-state
             :variant "outlined"
             :label "活动标题"
             :placeholder "Placeholder"
             :style {:width 400 :padding 10}
             :helper-text (str "An old " @race " proverb")
             ;;:class (.-textField classes)
             :on-change (fn [e]
                          (reset! text-state (.. e -target -value)))
             :inputRef #(js/console.log "input-ref" %)}]]
          [:div {:style {:margin-bottom 20}}
           [text-field
            {:value @text-area-state
             :label "活动内容详情"
             :placeholder "Placeholder"
             :helper-text "Helper text"
             ;;:class (.-textField classes)
             :on-change (fn [e]
                          (reset! text-state (.. e -target -value)))
             :multiline true
             ;; TODO: Autosize textarea is broken.
             :style {:width 400 :padding 10}
             :rows 10}]]
          [:div {:style {:margin-bottom 20}}
           [text-field
            {:value @text-state
             :label "地点"
             :placeholder "Placeholder"
             ;;:class (.-textField classes)
             :on-change (fn [e]
                          (reset! race (if (= (.. e -target -value) 2) "Sith" "Jedi"))
                          (reset! text-state (if (= (.. e -target -value) 2) @text-sith-state @text-jedi-state)))
             :select true}
            [:> mui/MenuItem
             {:value 1}
             "Lightside"]
            ;; Same as previous, alternative to adapt-react-class
            [:> mui/MenuItem
             {:value 2}
             "Darkside"]]]]
         ]
        ))))
        
(defn app []
  [demo-text-field {}])

(reagent/render [app]
                  (.getElementById js/document "app"))
                  
```

### [React-sortable-hoc](https://github.com/reagent-project/reagent/blob/master/examples/react-sortable-hoc/src/example/core.cljs)

    

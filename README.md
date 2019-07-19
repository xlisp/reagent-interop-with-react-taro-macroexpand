# reagent-interop-with-react-taro


* [Reagent Interop With React](http://reagent-project.github.io/docs/master/InteropWithReact.html "Permalink to Interop with React")
* [Taro](https://github.com/NervJS/taro)
* [Reagent demo](https://github.com/reagent-project/reagent/tree/master/demo), [Reagent doc](https://github.com/reagent-project/reagent/tree/master/doc), [Reagent examples](https://github.com/reagent-project/reagent/tree/master/examples)

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

## Creating React Components from Reagent "Components"

The `reagent/reactify-component` will take a Form-1, Form-2, or Form-3 reagent "component". For example:
    
``` clojure
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

    

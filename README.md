# Overview

Package allows defining visual styles for UGUI based UIs in somewhat CSS inspired manner, but way more specific to Unity/C#. Main features are as follows:

- Basic concepts are similar to ones in CSS (rules, selectors, property blocks, specificity, classes, ids).
- Style definition language is a C# script though.
- You don't need many extra components in your UI to make it styled.
- Can be either editor-time or runtime.


## Code-based

Main decision behind this package was how and where to define the style. First of all it was decided to keep it text based, because editing large style sheets in Unity editor inspector would inevitably become painful.

Second part of decision was to choose the style format. One of priorities was to avoid parsing some CSS or Json where developers would write types of Unity specific components and properties. That would be prone to typos, limiting in support. So, it was decided to remove any indirections and leverage the power of developer IDE with C# compiler to "Spell-Check"/"Auto-Complete" the style sheet as it's being created.

Style-sheet code is essentially an extension of ScriptableObject and will consist of rules like this:

    [CreateAssetMenu] public class MyStyleSheet : StyleSheet 
    {
        protected override List<Rule> GenerateRules() => new()
        {
            For<Text>().Set(text =>
            {
                text.color = Color.red;
                text.fontSize = 36;
                text.alignment = TextAnchor.MiddleCenter;
            }),

            In<Button>().For<Text>().Set(text =>
            {
                text.color = Color.blue;
            }),

            // and so on
        };
    }

With this kind of style format, you get the following:

- Instant validation of the style sheet. Most IDEs will complain about syntax or incorrect class or property names as soon as you type the wrong character.
- Auto-complete for all class/property names.
- Ability to "auto-complete" the whole pages of rules with AI-based code assistants.
- Ability to write plain C# code in property blocks to calculate the values for properties in extremely flexible ways. Not to be abused, though.
- And of-course, even though it may sound controversial, all benefits of text editing compared to mouse operated UI.


## Clean UI game objects

Another important aspect is that you don't need to assign the style component to all of your game objects. This way you can quickly draft your UI using default look of controls or copy and paste pieces of UI from previous projects, and then apply the correct style to the resulting scene.

In the simplest case all it takes to apply the style sheet to the UI is to add the Style component to the root object of your hierarchy and select the style-sheet (or multiple style-sheets) to be used.


## Editor-time and Runtime options

There are two main options how to apply styles:

- First option you have is to switch your style components' `dynamicity` to `None` and just apply styles in the Editor.
    - ✅ best performance, since styling engine is not involved in runtime at all.
    - ❌ If you have multiple scenes / prefabs with UI elements, you can forget to apply updated styles in some of them after changing the style sheet. Could be annoying.
- Second Option is the runtime application of styles (`dynamicity` set to  `OnChange`).
    - 🟡 Slower initialization, as the styles will be applied in OnEnable.
    - 🟡 Slower UI prefabs instantiation for example when you populate some list view.
    - ✅ Styles will apply automatically, so it's not necessary to apply them in editor. Although, you still can do that too for preview purposes.

And then, in case it's not enough, and you need to apply styles in every frame, you can set style `dynamicity` to `OnUpdate`. Even though style application code was optimized to avoid unnecessary operations and GC allocations, still for large UIs it can drop, if not tank, the performance. So, be careful with that option. 

> 🤓 **Tip:** There's an option to apply styles on Update only for the sub-set of UI. You can have a Style component on the root of your UI with dynamicity set to `OnChange`. And then somewhere deep in the hierarchy, you can have a control that requires dynamic styling, with the empty Style component and dynamicity set to `OnUpdate`. Child style will inherit all parent style sheets, but will be applied to it's sub-hierarchy on every frame.


# Noteworthy aspects

First of all, if some element matches multiple rules, property blocks for all of them will get executed, since they are just C# code. Their execution will happen in the order of specificity, so the most generic rule will apply first and the most specific the last. That ensures that more specific values will be applied in the end, but it may involve a sequence of redundant assignments. If you're planning for anything unusual in code blocks, be aware of that.

Second thing is that you can't cancel property assignments in the more specific rules, if they already happened in the more generic ones. You can assign some new value that will revert the change, but there's no "unset" feature.

Style sheets are inherently more static so to say.

# Potential work-flows

## "Non-styling" actions in style sheets

You are very much not limited in what can be done in the "property-blocks" of the style. The obvious usage is to assign colors, sprites, and other properties of UI components, but you can also do the following:
- perform all kinds of calculations on assigned values.
- modify non-UI components and otherwise initialize your scenes.
- create/instantiate more game objects if just decorating the existing stuff is not enough.

Although you should probably be careful with the last one to not create infinite loops and to not spam your scene with unwanted objects.

## Themes

Then, there's no particular enforced way of where to get the values to be assigned to properties. You can hardcode the values in the style sheet, like in the example above. Or you can use the fields of the style-sheet scriptable object to store values. This way you can create multiple instances of the same style-sheet with different values:

    [CreateAssetMenu] public class ThemedStyleSheet : StyleSheet 
    {
        [SerializeField] Color backgroundColor = Color.white;
        [SerializeField] Color foregroundColor = Color.black;

        protected override List<Rule> GenerateRules() => new()
        {
            For<Image>().Set(image =>
            {
                image.color = backgroundColor;
            }),

            For<Text>().Set(text =>
            {
                text.color = foregroundColor;
            }),
        };
    }

If your style components in scenes are set to update on change, you can switch styles and themes while app is running too.

## Global Styles

There's an option to add the GlobalStyle asset to Resources folder, where you can select style sheets that will be active even when Style components are empty. Can be useful if you have many scenes / prefabs with Style components and don't want to re-assign stylesheets in all of them if changes are rather often.

# Usage Guide

## Key Elements

*Style Sheet* - asset of StyleSheet type.

*Theme* - instance of the same StyleSheet type but with custom field values.

*Style* - Component added to the game object where you specify which style sheets are applied recursively to this game object and its children. Can be nested.

*Rule* - single building block for style definition. In simplest form consists of selector and property block.

    For<Text>().Set(text =>
    {
        text.color = Color.blue;
    })

You can have multiple property blocks per selector:

    In<Button>().For<Text>().Set(text =>
    {
        text.color = Color.blue;
    })

You can have multiple selectors that have the same group of property blocks applied to all of them:

    On<InputField>().For<Image>().
    On<Dropdown>().For<Image>().
    Set(image =>
    {
        image.SetColor("#222222");
    }).
    Set<RectTransform>(rt =>
    {
        rt.SetHeight(40);
    }),

*Property Block* - Action performed on the target component, assigned to the rule, using ```Set```, ```Set<T>``` or ```SetOrAdd<T>``` methods.

*Selector* - Chain of method calls that define a location of selected element relative to the other components. The only required segment of selector is ```For<T>``` it means that the property blocks following such selector will be applied to all objects containing component of type T.
Optionally For<T> can be prepended with other segments making the selection more specific. Theses are:

```On<P>().For<T>()``` - applies for T if it's located on the game object also having P.
```In<P>().For<T>()``` - applies for T if it's an immediate child of game object having P.
```DeepIn<P>.For<T>()``` - applies for T if it's a child of game object having P.
```Near<P>.For<T>()``` - applies for T if it has a sibling game object having P.
```From<P>.For<T>(Func<P, T>)``` - applies for T that is resolved from P using a given function. Example is: ```From<Button>().For<Image>(b => b.TargetGraphic)```

*Predicates* - it's possible to make selectors more specific specifying predicates that will filter the components used in selector. In it's generic form predicates are added using a That() or Where() method which follows the selector segment. Special case is Named() method for checking the game object name. Examples are:

    For<Image>().Where(i => i.sprite == null).Set(i => 
    {
        i.color = Color.black
    })

    DeepIn<Canvas>().That(c => c.pixelPerfect).For<Image>(i => 
    {
        i.color = Color.red;
    })

    For<Image>().Named("Background").Set(image =>
    {
        image.preserveAspect = true;        
    })


*Helper Extensions*

To keep rule selectors and property blocks more compact there's a number of extension methods available, and you can add your own as needed. They can be found in files Extensions.cs, Predicates.cs, Resolvers.cs and Setters.cs. Some predicate examples:

    IsFirst
    IsFirst(n)
    IsLast
    IsLast(n)
    IsNth(n)
    IsNth(n, offset)
    IsOdd
    IsEven

Setter helpers:

    Color Blend(this Color, Color color, float amount)
    Color Dim(this Color, float amount) // Blends with black
    Color Saturate(this Color, float amount) // Blends with white
    Color Fade(this Color, float amount) // Blends with transparent
    void SetColor(this Graphic, string hexColor)
    void SetColor(this Graphic, Color color)
    void SetAlpha(this Graphic, float alpha)
    void SetSprite(this Image, string spriteName)
    void SetAspectRatio(this AspectRatioFitter, string spriteName)
    void SetAspectRatio(this AspectRatioFitter, Sprite sprite)
    void Align(this RectTransform, Alignment alignment)
    void HAlign(this RectTransform, float min, float max)
    void VAlign(this RectTransform, float min, float max)
    void SetX(this RectTransform, float x)
    void SetY(this RectTransform, float y)
    void SetWidth(this RectTransform, float width)
    void SetHeight(this RectTransform, float height)
    void SetBorder(this RectTransform, float border)
    void SetTop(this RectTransform, float border)
    void SetBottom(this RectTransform, float border)
    void SetLeft(this RectTransform, float border)
    void SetRight(this RectTransform, float border)
    

*Markdown-inspired predicates*

Helper extensions include detection of name based element categories. If your game object name starts with a "# ", for example "# Title", then it's considered to be level 1 header. Game object starting with "- " is considered a list item, etc. It's somewhat similar to markdown syntax. Full list is as follows:

    IsH1 - "# Level 1 Header"
    IsH2 - "## Level 2 Header"
    IsH3 - "### Level 3 Header"
    IsH4 - "#### Level 4 Header"
    IsH5 - "##### Level 5 Header"
    IsH6 - "###### Level 6 Header"
    IsQuote - "> Quote"
    IsListItem - "- List Item" or "+ List Item" or "* List Item"

With all of these helpers your rules can look like this:

    For<Text>.That(IsH1).Set(text =>
    {
        text.SetAlpha(0.5f);
    })

    For<RectTransform>.That(IsFirst).Set(rt =>
    {
        rt.SetBorder(5);
    })

    From<Button>().For<Image>(TargetGraphic).Set(image =>
    {
        image.SetColor("#003B7A");
    })

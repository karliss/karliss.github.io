---
layout: post
title:  "OSM indoor mapps in OsmAnd"
date:   2026-09-27 23:00:00 +0300
tags: maps software
---

Recently I have been working on OsmAnd theme for indoor mapping.

While I don't think the current approach can be extended to proper indoor support without significant changes on OsmAnd side, it's nice to have even something half working to avoid chicken and egg problem. Software doesn't add support indoor mapping because there are few buildings with indoor tags. There are few buildings with indoor tags because there is hardly any software for displaying it all.

What works:
* drawing of indoor=room;area;corridor 
* basic level filtering (only simple level=x assignments)
* level filtering for rooms, highway=footway;corridor; some POI

Limitation of current solution:
* level ranges or list level=x;y or level=x-y are not supported (could be improved by minor change in OsmAnd .osm->.obf processing tool without changing OsmAnd itself or data format)
* repeat_on is not supported
* indoor features with unrecognized level are always displayed (good enough for most common cases of stairs and elevators that go through whole building)
* POI filtering applies only to some common types of objects you would expect to find in indoor mapped malls and train stations like shops, cafes and amenities (not even all of them, only specific kinds)
* no concerns for interaction with layers, although doesn't seem like it's well documented how those 2 osm tags should interact
* for a long time OsmAnd map files didn't include level=2. Levels <=1 and >=3 are fine, but 2 was missing due to bug. It was recently fixed, but it might take a while until updated map files become available.


Overall creating a custom OsmAnd theme was somewhat frustrating both due to lack of documentation and OsmAnd rendering rules behaving in unintuitive way which is unfriendly to certain kind of modifications.

From my testing current state works somewhat usable with shopping mall/university type buildings. Complex train and metro stations, which use combination of layers and levels, have parts which overlap with outdoor roads and other areas, which have stairs and escalators all over the place (different places at different levels) don't work as good.


Theme can be downloaded [here](https://github.com/karliss/osmand-indoort)


# Basic introduction to how OsmaAnd themes works

There are 2 main pieces of documentation https://osmand.net/docs/technical/osmand-file-formats/osmand-rendering-style/ and comments at the start of default theme default.renderer.xml.

The following is not an official documentation, it can contain major mistakes. This is my best understanding after spending a few days bashing head against wall and trying to figure out why the rules don't work the way I expected.

The structure of OsmAnd theme looks roughly like this

```xml
<!-- Theme constants and properties -->
<rendederingConstant />
<renderingProperty />

<order>
    <!-- Order rules -->
</order>

<polygon>
    <!-- Polygon feature style rules -->
</polygon>
<line>
    <!-- line feature style rules -->
</line>
<point>
    <!-- point feature style rules -->
</point>
<text>
    <!-- text label rules -->
</text>
```

Constants are useful for defining fixed reusable values like exact colors or minimum/maximum zoom levels.

Rendering properties are for user configurable values, enabling optional theme features or changing modes.

The documentation describes the rule system as simply evaluating rules top to bottom, but it's a lot more complicated than that. There plenty of critical details which break the simple mental model of top to bottom with later rules overriding earlier ones.

Before any rule evaluation there is question of what OSM data is available to theme. Many of OSM tags are available as inputs for rule filters but it's not exact and exhaustive 1:1 mapping. For the purpose of efficiency and normalization OSM data needs to be processed by OsmAnd specific tool. Only specific tags listed in configuration are available, some tags are renamed or values modified. Current tag processing rules can be found in rendering_types.xml https://github.com/osmandapp/OsmAnd-resources/blob/master/obf_creation/rendering_types.xml .
So if you where thinking of making a custom addon theme for visualizing a niche map feature for rarely used tags, there is a high chance it will not work unless you prepare your own map files.


Rendering rule processing happens in multiple stages. First the order rules determine which objects to draw, what order and whether they should be categorized as point, line or polygon.


## Rule evaluation

All rule categories are based on same evaluation engine and look roughly like this.

```xml
<switch>
    <case input1="v1" input2="v1" input3="v3" output1="o1"/>
    <case input1="v2" input2="v2" output2="o2"/>
    <switch input1="v1" output="v3">
        <case ... />
    </switch>
    <apply output4="o123/">
    <apply input5="v5" output5="o5"/>
    <apply_if input6="v6" output6="o6"/>
</switch>
<switch input1="i1">
    <case input2="v1" output3="1">
    <case input2="v2" output5="2">
</switch>
```

When reading default theme for the first time it can be easy to get confused which parts of the rules are input conditions and which are the outputs. Only way to distinguish them is by reading documentation of each attribute.

Rules are evaluated top to bottom checking for input conditions. Nested scopes are processed only if parent condidtions are true. When a terminal case is successfully matched it performs early return stopping further evaluation except  .

Apply rules get executed only when parent switch matches any of the cases. There is no difference between `<apply>` and `<apply_if>` they are alias to each other, both can contain input conditions. You can have `<apply>` with additional input conditions, you can have `<apply_if>` with no input conditions and only output values. When reading for the first time it might seem that first attribute inside `<apply_if>` is an input parameter, but it's not guaranteed. Whether something is input or output is determined by the parameter itself not the placement within `<apply>` or `<apply_if>` tags.

So in the example above after matching `input1=v2 input2=v2` will set `output2=o2; ouptut4=o123` and possibly `output5=o5; output6=o6;` depending on input5 and input6. Second top level switch will not be evaluated. Same example with `input1=i1; input2=v1` will set `output3=1` and not evaluate any of apply statements from first top level switch. 


Common input attributes
* all renderingPropeties
* minZoom/maxZoom for rendering something only when you are sufficiently zoomed in or zoomed out
* tag, value correspond to osm tag names and values (after remapping and filtering by render_types.xml)
* additional='key=val', additional osm tag value checks

Each terminal case needs a exactly one unique tag="??" and value="??" pair somewhere in the chain up to root tag. Doesn't matter where exactly in the hierarchy you put the tag and value checks but you need to have them. You can have any amount of additional tag checks.

This means you can't do 

```xml
<switch tag="highway" value="footway">
    <case tag="surface" value="asphalt" color="#12345"/> <!-- more than one tag/value pair in parent/child chain --> 
</switch>
```

but you are allowed to do

```xml
<switch tag="highway" value="footway">
    <case additional="surface=asphalt" color="#12345"/>
</switch>
<switch tag="highway" additional="access=private">
    <case value="cycleway" color="#555555"/>
    <switch value="path">
        <case additional="surface=paving_stones" color="#ff0000"/> <!-- highway=path + access=private + surface=paving_stone  multiple additional but one main tag/value pair-->
        <case additional="surface=unpaved" color="#ffff00"/>
    </switch>
</switch>
```

Empty tag="" and value="" are wildcards which match any values, but their evaluation order doesn't match sequential top do bottom model with early exit. First the system checks exact tag/value match, then it checks tag/"" and finally if previous steps
 didn't match anything it checks ""/"".
 
Example:

```xml
<switch>
    <case tag="highway" value="" additional="access=yes" color="#aaaaaa">
    <case tag="highway" value="footway" additional="access=private" color="#bbbbbb">
    <case tag="highway" value="footway" color="#cccccc">
<switch>
```
When checking against "highway=footway;access=yes" will match the third case "color=#cccccc" even though it's later in the top to bottom evaluation chain. "highway=cycleway;access=yes" will match the first rule. 
You wouldn't explicitly write rules like that, but it can easiliy happen when creating a derived theme. 
If the base theme contains more specific rule `tag=x value=y` the derived theme can't override it using wildcard rule `tag=x value=""`. 

Some useful patterns:

Common properties using apply, output1=v1 will be applied to all three kinds of objects x1,x2, x3. 
```xml
<switch>
    <case tag="x1" value="y1"/>
    <case tag="x2" value="y2" output0="o0"/>
    <case tag="x3" value="y3"/>
    <apply output1="v1"/> <!-- common -->
    <apply_if additional="extra" output2="v2"/> <!-- conditional common -->
    <apply_if additional="extra2" output2="v3"/> <!-- conditional common -->
    <apply_if additional="extra3" output3="v4"/> <!-- conditional common -->
</switch>
<switch>
    <case tag="x4" value="y4" output2="v2"/> <!-- apply from sibling switch has no effect here -->
</switch>
```



Almost `(x||y||z)&&(a||b||c)`


```xml
<switch>
    <case tag="x1" value="x"/>
    <case tag="x2" value="y"/>
    <case tag="x3" value="z"/>
    <apply>
        <switch>
            <case additional="e=a"/>
            <case additional="e=b"/>
            <case additional="e=c"/>
            <apply ouptut1="v1"/>
        </switch>
    </apply>
</switch>
<switch>
    <case tag="x1" value="x" additional="e=d" output1="v2"/> <!-- This case will never trigger since x1=x was already and matched in first switch. Lack of match in apply doesn't change that x1=x was already consumed. -->
</switch>
```

## Tag/value vs additional

From the first glance it's not fully obvious what the differences between matching tags using `tag="key"` and `value="value"` compared to `additional="key=value"`. As mentioned previously a rule matches single exactly single tag/value pair when taking into account parent scopes, no more no less.

You can have multiple additional tag checks (not directly on single tag xml rules still apply), but it cannot be used to bypass tag/value limitation. Only keys marked as `additional="true"` in render_types.xml can be used in additional checks.

For example if you have `{building=retail, shop=clothes, clothes=children, access=permissive}`. It can be matched with rules `{tag=building,value=retail, additional="access=permissive"}`, `{tag=shop,value=clothes,additional="clothes=children"}` or `{tag=shop,value=clothes,additional=[clothes=children, access=permissive]}`, but you cannot match it using rules `{tag=building, additional="shop=clothes"}` or `{tag=access, value=permissive, additional="building=retail"}`. Building and shop are not considered additional tags by rendering_types.xml and access isn't a main tag.

Usually main tags will roughly correspond to top level osm tags but it's not hard rule, main deciding factors are rendering_types.xml and needs of built in themes.

Some additional information on main/additional tags https://osmand.net/docs/technical/map-creation/create-offline-maps-yourself/#custom-vector-map-tags .


Putting it all together processing of objects look something like this, where get_rule iterates over the rules top to bottom with early exit.

```c
for (osmobj in bounding_box) {
    for (key,value in osmobj.main_tags) {
        if (order, obj_type = match_rules(key, value, osmobj.additional_tags)) {
            add_draw_obj(osmobj, order, obj_type);
        } else if (order, obj_type = match_rules(key, "", osmobj.additional_tags)) {
            add_draw_obj(osmobj, order, obj_type);
        } else if (order, obj_type = match_rules("", "", osmobj.additional_tags)) {
            add_draw_obj(osmobj, order, obj_type);
        }
    }
}
```
This implementation detail is responsible for breaking mental model of simple top to bottom evaluation order when using wildcards or objects with multiple main tags.


## Order

Order stage roughly determines which objects to draw and in what sorting order. It has 2 output parameter objectType and order.


objectType 1=point, 2=line, 3=polygon 

Object type determines whether object should be rendered as point line or polygon. Point is can have an icon with label (described in point and text sections). Line is for features like roads, paths and simple walls with the style described in line and text section. 
Polygons infill, outline stroke and they can also have icon and label. Slightly weird part is that icon for polyon is described in point section, but the outline stroke in the polygon section together with infill instead of line section. It would be more consistent if either the object type matched 1:1 to styling sections, or they objects where fully decomposed meaning that polygon objects read style from polygon, line and point sections.

|objectType\rule section|point|line|polygon|text|
|---|---|---|---|---|
|1(point)|✔️|||✔️|
|2(line)||✔️|||
|3(polygon)|✔️||✔️|✔️|


Order is an integer 0-255 or -1

Order determines layering of lines and polygons. But the point icons and labels have additional ordering rules. -1 is a special value meaning do not render.


In theory the default theme order values are roughly allocated 1-10 polygons, 11-100 lines, 101-200 points. But it's not hard to find tags where this assignment is violated. For example building=yes+layer=3 is using order=13. Yet some of the rendering logic groups drawable objects into polygons/lines based on DEFAULT_POLYGON_MAX=11.

No idea what exactly order does for points, since there is also separate iconOrder, distance/overlap based system for deciding which of the points/labels display when there are a lot of them close to each other, which doesn't behave quite like simple layering system.

Single OSM object can produce multiple render objects based on main tags. For example `{building=retail, shop=clothes}` can create polygon object from building=retail and point object from shop=clothes. In some ways this means that there is little difference whether shop was tagged as separate node inside building or on the same object as building. Due to limitation of how main/additional tag matching works you can't differentiate these two tagging approaches even if you wanted to. 

When writing styling rules for such objects you have to consider whether you are referring to point/text produced by building tag or the shop even if it all comes from single osm object. 


## Polygon, Line

Once the object type and drawing order is determined, OsmAnd runs the corresponding line or polygon rules to determine styling properties. Main style output properties are color, color_2, ... color_n and stroke_width, stroke_width_2, ... . 
Additional color values allow making layered multi color strokes. The order is color__2 (-2), color__1 (-1), color, color_2, color_3, ... . Double underscore means negative values. Simply color is the base color in case of polygons thats the infill, with color_2 being first stroke color. But for the lines color is the first stroke color. Similar naming scheme applies to other properties like strokeWidth, pathEffect. There is no strokeWidth without number for polygons since the base color is infill instead of stroke. PathEffect and hmargin can be used for dashed and offset strokes in more complex layered stroke styles.

Only objects with non negative order are considered.

First rule which matches performs early exit preventing further rule evaluation (except apply rules in parent scopes).


## Point

Describes point object icons.

## Text

Text section has a couple unique parameters which don't behave like most parameters in other rules.

Documentation clasifies `nameTag` as input parameter, but instead boolean simple check whether rule applies it determines which tag values (name, brand, addr:housenumber) to use as label text. 

Text rules also have `textOrder` which in combination with overlap avoidance determine which labels to show. Lower textOrder means it's more likely to show, opposite from polygon order.

A match on tag/value/nameTag doesn't prevent match on the same tag/value with different nameTag. If you want to override a label, it might be necesarry define a rule for each tag/value/nameTag triplet used by base theme.


# Challanges of creating custom theme

It is relatively easy to create a theme which only customizes colors by changing rendering constants.

Full custom theme is possible but take a lot of work, not very practical if you want to only customize certain aspects of map rendering. The default builtin theme is 12k lines of XML code.

The middle ground of themes which make nontrivial changes in rendering behavior, but doen't need to redefine everything is where OsmAnd theme system becomes difficult to use.

First catch when using the derived/base theme feature is that derived theme rules are listed **before** the base theme rules. This means that instead of overriding values set by base theme rules, derived theme sets the properties for objects it's interested in which blocks base theme from touching them.

Another problematic part of using derived themes is tag wildcard priority. If the base theme defines exact tag/value pairs you can't use wildcards to partially adjust style or even for hiding groups of same category objects. You need to list all the key/value pairs base theme listed which can be a lot for shops and amenities, and in case of styling extra it means potentially duplicating large part of base theme logic.

When creating a derived theme base theme constants and properties aren't automatically inherited. They need to redefined.


# Wish list for OsmAnd theme system improvements

Mechanism for overlay themes which can be freely combined by user. Current derived theme system requires single specific base theme, which means that if you create an overlay theme user can't choose base color theme and user also can't enable more than one custom overlay theme. There are some built in overlays which can be activated independently, but they need to be defined in default base theme.

Add on theme situation is made worse by not having mechanism which partially overwrites only some of previously set properties. When a derived theme rules match an object early return blocks base theme from affecting the same object. It's hard to change it without major redesign of whole rule engine. 
Within single theme this problem is partially avoided by having apply rules which get executed sequentially after terminal rule match is hit. Output properties from multiple apply statements get combined. Potential backwards compatible solution would be to remove restriction on top level apply statements and listing top level apply statements in reverse order.

```xml
<theme name=overlay1>
    <polygon>
        <switch overlay1_condition>
            <case oc1/>
            <apply oc2/>
        </switch>
        <apply_if overlay1_cond=c1 overlay1_output=o1/>
    </polygon>
</theme>
<theme name=overlay2>
    <polygon>
        <switch overlay2_condition>
        </switch>
        <apply_if overlay2_cond=c2 overlay2_output=o2/>
    </polygon>
</theme>
<theme name=color_theme base=default_theme>
    <!-- mostly rendering constant changes -->
    <polygon>
        <switch color_theme_c3>
        </switch>
        <apply_if colort_cond=c3 colort_output=o3/>
    </polygon>
</theme>
<theme name=default_theme>
    <polygon>
        <switch base_theme_cond>
        </switch>
        <apply_if base_cond=c1 base_output=o1/><!-- Little need for base theme top level apply, since the base theme can put them in narrower scopes. -->
    </polygon>
</theme>
```

Could be combined into

```xml
<combined_theme>
    <polygon>
        <switch overlay1_condition>
            <case oc1/>
            <apply oc2/>
        </switch>
        <switch overlay2_condition>
        </switch>
        <switch color_theme_c3>
        </switch>
         <switch base_theme_cond>
        </switch>
        <apply_if base_cond=c1 base_output=o1/>
        <apply_if colort_cond=c3 colort_output=o3/>
        <apply_if overlay2_cond=c2 overlay2_output=o2/>
        <apply_if overlay1_cond=c1 overlay1_output=o1/>
    </polygon>
</combined_theme>
```

It could work but root level apply statements have potential performance issues if done naively since all of them would be evaluated for each displayed object. It might be possible to mitigate the performance concerns by smarter rule filtering instead of recursively evaluating them as written in xml files.
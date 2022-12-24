Godot 4.0-Beta 10's editor seems to have a bug when saving Control nodes that
use the "full rect" anchor preset.  It causes spurious changes to be "randomly"
written to the .tscn file, resulting in version control chaos.

Well, it doesn't seem to be _completely_ random.  It appears it has something to
do with closing and opening tabs before saving.  The simplest way to reproduce
it is this:
* Create a blank Control scene named `TitleScreen.tscn`
* Commit your change in git
* Close the tab for `TitleScreen.tscn`
* Open a new tab for `TitleScreen.tscn`
* Re-save `TitleScreen.tscn`, without making any changes
* Look at the git diff.  You'll notice that two new lines have appeared, despite
    no changes having been made.

To help narrow down what's going on, I've done some tests and listed my notes
here.  Each test also has a branch in this repo, so you can follow along with
the commit history with my notes.

## Test A: single file, single label, center of screen, closed and re-opened
* Create a Control scene named TitleScreen, with a single label as a child.  Drag
    the label to approximately the center of the screen.
    ```tscn
    [gd_scene format=3 uid="uid://bkhsg8tfwqc74"]
    [node name="TitleScreen" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0

    [node name="Label" type="Label" parent="."]
    layout_mode = 0
    offset_left = 399.0
    offset_top = 154.0
    offset_right = 439.0
    offset_bottom = 177.0
    text = "Hello world"

    ```
* Close the TitleScreen tab, open it, and re-save it.  It will gain a
    `grow_horizontal = 2` line and a `grow_vertical = 2` line.
    ```tscn
    [gd_scene format=3 uid="uid://bkhsg8tfwqc74"]

    [node name="TitleScreen" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0
    grow_horizontal = 2
    grow_vertical = 2

    [node name="Label" type="Label" parent="."]
    layout_mode = 0
    offset_left = 399.0
    offset_top = 154.0
    offset_right = 439.0
    offset_bottom = 177.0
    text = "Hello world"

    ```

* Close the TitleScreen tab, open it, and re-save it.  It will not change.

## Test B: Just the root, no label, closed and re-opened
* Create a blank Control scene named TitleScreen.  Do not close, modify, or save it.
    ```tscn
    [gd_scene format=3 uid="uid://csvc51eb4l38u"]

    [node name="TitleScene" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0

    ```
* Save it.  Nothing changes.
* **Bug**: Close the TitleScreen tab, re-open it, and then re-save it.  It gains 2 lines.
    ```tscn
    [gd_scene format=3 uid="uid://csvc51eb4l38u"]

    [node name="TitleScene" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0
    grow_horizontal = 2
    grow_vertical = 2

    ```
* Close it again, re-open it again, and then re-save it again.  No change.
### Uninteresting bit
* Create a new Label node, and save it.  Only the expected changes occur.
    ```tscn
    [gd_scene format=3 uid="uid://csvc51eb4l38u"]

    [node name="TitleScene" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0
    grow_horizontal = 2
    grow_vertical = 2

    [node name="Label" type="Label" parent="."]
    layout_mode = 0
    offset_right = 40.0
    offset_bottom = 23.0

    ```
* Save it.  Nothing changes.
* Close it again, re-open it again, and then re-save it again.  No change.
* Click the "reset" icon on the root node's `Anchors Preset` field in the inspector, then save.
    ```tscn
    [gd_scene format=3 uid="uid://csvc51eb4l38u"]

    [node name="TitleScene" type="Control"]
    layout_mode = 3
    anchors_preset = 0

    [node name="Label" type="Label" parent="."]
    layout_mode = 0
    offset_right = 40.0
    offset_bottom = 23.0

    ```
* Save it.  Nothing changes.
* Close it again, re-open it again, and then re-save it again.  No change.

## Test C: Paying more attention to `Anchors Preset`
* Create a blank Control scene named TitleScreen.  Do not close, modify, or save it.
    ```tscn
    [gd_scene format=3 uid="uid://73e5lurkok6x"]

    [node name="TitleScreen" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0
    ```
* Click the "reset" icon on the root node's `Anchors Preset` field in the inspector, then save.
    ```tscn
    [gd_scene format=3 uid="uid://73e5lurkok6x"]

    [node name="TitleScreen" type="Control"]
    layout_mode = 3
    anchors_preset = 0

    ```
* Close the tab, re-open it, and re-save it.  Nothing changes.
* **Insight**: This bug only occurs when `Anchors Preset` is left the way it
    starts (which is _not_ the same as its default value!)

## Test D: What if we modify and save the scene before closing and re-opening?
* Create a blank Control scene named TitleScreen.  Do not close, modify, or save it.
    ```tscn
    [gd_scene format=3 uid="uid://cytfc5cxv7a6h"]

    [node name="TitleScreen" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0

    ```
* Rename the root node to `TitleScreenFoo` and save it (without closing it).
    Only the expected change occurs.
    ```tscn
    [gd_scene format=3 uid="uid://cytfc5cxv7a6h"]

    [node name="TitleScreenFoo" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0

    ```
* **Bug**: Close the tab, re-open it, and re-save it.  It gains 2 lines.
    ```tscn
    [gd_scene format=3 uid="uid://cytfc5cxv7a6h"]

    [node name="TitleScreenFoo" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0
    grow_horizontal = 2
    grow_vertical = 2

    ```
* **Insight**: Changes made before closing the tab work as expected.  There must
    be something special about the tab opened for newly-created scenes.  Perhaps
    they have some kind of state that re-opened tabs don't?

## Test E: Do children with the same anchor preset have the same problem?
* Create a blank Control scene named TitleScreen.  Do not close, modify, or save it.
    ```tscn
    [gd_scene format=3 uid="uid://bt43ed66tovuy"]

    [node name="TitleScreen" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0

    ```
* Add some children, one of which uses "Full Rect" for its `Anchors Preset`, and
    and the other using "Top Left".  Then save it.  Notice that the full-rect
    child we added already has `grow_horizontal` and `grow_vertical` lines.
    ```tscn
    [gd_scene format=3 uid="uid://bt43ed66tovuy"]

    [node name="TitleScreen" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0

    [node name="FullRectAnchors" type="Control" parent="."]
    layout_mode = 1
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0
    grow_horizontal = 2
    grow_vertical = 2

    [node name="TopLeftAnchors" type="Control" parent="."]
    layout_mode = 1
    anchors_preset = 0
    offset_right = 40.0
    offset_bottom = 40.0

    [node name="PositionalLayoutNoAnchors" type="Control" parent="."]
    anchors_preset = 0
    offset_right = 40.0
    offset_bottom = 40.0

    ```
* **Insight**: This suggests that those two lines really are "supposed" to be
    there.  Perhaps the editor is forgetting to add them when initially creating
    the file?  But then why doesn't re-saving it cause them to appear until
    _after_ the tab is closed and re-opened?
* Close the tab, re-open it, and re-save it.
    * **Bug**: The root node gains those two lines
    * **Bug**: The node using "Position" layout gets changed to "Anchors".
    ```tscn
    [gd_scene format=3 uid="uid://bt43ed66tovuy"]

    [node name="TitleScreen" type="Control"]
    layout_mode = 3
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0
    grow_horizontal = 2
    grow_vertical = 2

    [node name="FullRectAnchors" type="Control" parent="."]
    layout_mode = 1
    anchors_preset = 15
    anchor_right = 1.0
    anchor_bottom = 1.0
    grow_horizontal = 2
    grow_vertical = 2

    [node name="TopLeftAnchors" type="Control" parent="."]
    layout_mode = 1
    anchors_preset = 0
    offset_right = 40.0
    offset_bottom = 40.0

    [node name="PositionalLayoutNoAnchors" type="Control" parent="."]
    layout_mode = 1
    anchors_preset = 0
    offset_right = 40.0
    offset_bottom = 40.0

    ```
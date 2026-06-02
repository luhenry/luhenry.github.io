---
title: "Wrestling with Windows: Avoid the switch to AZERTY mid-sentence while Spell-Checking French"
layout: post
---

If you write in multiple languages on Windows, you've probably run into this frustrating scenario: you want the OS to autocorrect and spell-check a second language (like French). So, you go ahead and add it in the Settings. But Windows, being Windows, automatically attaches a new keyboard layout (like AZERTY) to it.

When you try to remove that layout to keep your default QWERTY keyboard clean, the "Remove" button is completely greyed out.

Let's look into why this happens and how to forcefully bypass it using a bit of PowerShell.

# The stubborn "Remove" button

Windows has a strict, underlying rule for language packs: a language cannot exist without at least one associated keyboard layout.

When you try to remove the only layout tied to a language, the UI blocks you. Damn, what's happening here? Well, Windows is just trying to protect you from entering an invalid state where a language has no input method.

If you try to bypass the UI and use PowerShell's `$lang.InputMethodTips.Clear()` to strip all keyboards, you might think you've won. But Windows will secretly restore the default layout on the next execution. It just refuses to let the language exist empty.

```powershell
PS C:\Windows\system32> Get-WinUserLanguageList

LanguageTag     : en-US
Autonym         : English (United States)
EnglishName     : English
LocalizedName   : English (United States)
ScriptName      : Latin
InputMethodTips : {0409:00000409}
Spellchecking   : True
Handwriting     : False

LanguageTag     : fr-FR
Autonym         : Français (France)
EnglishName     : French
LocalizedName   : French (France)
ScriptName      : Latin
InputMethodTips : {040C:0000040C}
Spellchecking   : True
Handwriting     : False
```
*Notice how `InputMethodTips` for `fr-FR` stubbornly holds onto `{040C:0000040C}` (the French layout) despite our attempts to clear it.*

# The PowerShell Judo Move

Since Windows insists that the French language pack *must* have a keyboard, we can just trick it. We'll assign your standard US keyboard layout to the French language, and *then* remove the French AZERTY layout.

This gives you the best of both worlds: French spell-checking remains active, but even if the OS accidentally toggles your active language to French, your physical keys will still output standard US English.

Let's do this via PowerShell. First, fetch the current language list:

```powershell
$LangList = Get-WinUserLanguageList
```

Next, let's grab the target language (French in our case):

```powershell
$lang = $LangList | Where-Object { $_.LanguageTag -eq "fr-FR" }
```

Now for the magic. We add the US English keyboard layout (`0409:00000409`) to the French language profile, and remove the unwanted French AZERTY layout (`040C:0000040C`):

```powershell
$lang.InputMethodTips.Add("0409:00000409")
$lang.InputMethodTips.Remove("040C:0000040C")
```

Finally, apply the changes forcefully:

```powershell
Set-WinUserLanguageList $LangList -Force
```

# Did it work?

Let's verify our setup:

```powershell
PS C:\Windows\system32> Get-WinUserLanguageList

LanguageTag     : en-US
Autonym         : English (United States)
EnglishName     : English
LocalizedName   : English (United States)
ScriptName      : Latin
InputMethodTips : {0409:00000409}
Spellchecking   : True
Handwriting     : False

LanguageTag     : fr-FR
Autonym         : Français (France)
EnglishName     : French
LocalizedName   : French (France)
ScriptName      : Latin
InputMethodTips : {}
Spellchecking   : True
Handwriting     : False
```

Under your `fr-FR` language tag, `InputMethodTips` now gracefully accepts an empty array (`{}`)! Most importantly, `Spellchecking` will remain `True`.

And that's it! That's how you keep your multilingual autocorrect without accidentaly switching to yout AZERTY keyboard mid-sentence.

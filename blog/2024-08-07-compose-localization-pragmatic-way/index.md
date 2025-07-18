---
title: Localization in Compose - The pragmatic way
slug: compose-localization-pragmatic-ways
tags: [compose, kotlin]
---

I have been thinking for a while regarding localization-as-code approach for Compose.
Currently, either in Jetpack Compose or Compose Multiplatform, we usually place the
localized strings in resource files. But I felt this approach has some limitations because
of these limitations:

* String resource files can hold plain strings or strings with placeholder notations, but
  it has limitations to define annotated strings.

* It has limitations to conditionally rephrase a string based on placeholder values. 
  Eg: plural expressions

* Since the resource files are XML, you might need to use `<![CDATA[]]>` during some instances
  to escape some special characters.

## The motivation

I like the way how we access the Material theme properties in Compose code. It is seamless,
independent, simple and concise.

```kotlin
Column {
    Text(
        text = "Hello, World!",
        style = MaterialTheme.typography.bodyMedium,
        color = MaterialTheme.colorScheme.primary,
    )
}
```
The Material theme properties are provided by `MaterialTheme` composable which is usually
placed in the top of the composable tree. It internally uses `CompositionLocalProvider` to
propogate the theme properties down through the hierarchy. Also when the provided object 
changes, it is very smart to recompose those specific parts of the tree where the properties
are used.

Inspired by the `MaterialTheme` in Compose, we can use the same approach for localization.
Propogate the data structure of the localized objects through a `CompositionLocalProvider`
placed on the root of the composable tree, and use it seamlessly deep down the hierarchy.
When the user wants to update the application locale, provide the updated localized objects
to the `CompositionLocalProvider` and let it recompose the parts of the hierarchy affected 
by the locale change.

## The implementation

Instead of keeping strings in separate resource files, we can keep them in the Kotlin code
itself. We can define an interface, provide properties or methods which return localized
strings.

```kotlin
interface DefaultStrings {
    companion object : DefaultStrings
    val greeting: String
        get() = "Hello, World!"

    fun apples(count: Int): String {
        return when (count) {
            1 -> "1 Apple"
            else -> "$count Apples"
        }
    }
}
```
> Kotlin allows to create a companion object which we can craft it to be a default instance
> that implements the same interface accessible by the name of the interface itself. (This is
> inspired from the Modifier in Compose).

For providing localization, we can override the `DefaultStrings` interface to a localized
version.

```kotlin
// Caution: These translations are provided by Google Translate. 
interface CzechStrings : DefaultStrings {
    companion object : CzechStrings

    override val greeting: String
        get() = "Ahoj světe!"

    override fun apples(count: Int): String {
        return when (count) {
            in 1..4 -> "$count jablka"
            else -> "$count jablek"
        }
    }
}
```

> Czech language is used in this sample code because the plural expression is different than
> English language, and well demonstrates how to tackle language specific plurals.

Once we have all the localized objects, we can define a `CompositionLocal` to propogate the
localization object through the hierarchy, and a global `MutableStateFlow` to update the locale
when needed. Also, we define a composable `LocaleProvider` to encapsulate the logic, and place
it in the top of the hierarchy.

```kotlin
val LocalStrings = compositionLocalOf<DefaultStrings> { DefaultStrings }
val AppLocale = MutableStateFlow("en")

@Composable
fun LocaleProvider(
    localeOverride: String? = null,
    content: @Composable () -> Unit,
) {
    val localeState: State<String>? = if (localeOverride == null) {
        AppLocale.collectAsState()
    } else {
        null
    }
    val locale = localeOverride ?: localeState?.value
    val strings = when (locale) {
        "cz" -> CzechStrings
        "ar" -> ArabicStrings
        "es" -> SpanishStrings
        "de" -> GermanStrings
        else -> DefaultStrings
    }
    val layoutDirection = when (locale) {
        "ar" -> LayoutDirection.Rtl
        else -> LayoutDirection.Ltr
    }
    CompositionLocalProvider(
        LocalStrings provides strings,
        LocalLayoutDirection provides layoutDirection,
        content = content
    )
}
```
In the above example, we consider the layout directionality along with the localization. This is
good when you localize the app to RTL languages like Arabic. It is also notable that `localeOverride`
parameter in `LocaleProvider` composable is provided to help with generating previews on other
locales.

## The demo

For the demonstration, let's build a simple composable UI with some texts and buttons. This example
will be demonstrating the Czech localization of the UI because Czech language has different plural
expressions than English language.

```kotlin
@Composable
fun AppleCounter(
    modifier: Modifier = Modifier,
) {
    Column(
        modifier = modifier,
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
    ) {
        Text(text = LocalStrings.current.greeting)
        var count by rememberSaveable { mutableIntStateOf(0) }
        Text(text = LocalStrings.current.apples(count))
        Row {
            Button(onClick = { count++ }) {
                Text(text = "+")
            }
            Button(onClick = { count-- }) {
                Text(text = "-")
            }
        }
        Row {
            Button(onClick = { AppLocale.value = "en" }) {
                Text(text = "en")
            }
            Button(onClick = { AppLocale.value = "cz" }) {
                Text(text = "cz")
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
private fun AppleCounterEnglishPreview() {
    MaterialTheme {
        LocaleProvider {
            AppleCounter(modifier = Modifier.size(300.dp))
        }
    }
}

@Preview(showBackground = true)
@Composable
private fun AppleCounterCzechPreview() {
    MaterialTheme {
        LocaleProvider(localeOverride = "cz") {
            AppleCounter(modifier = Modifier.size(300.dp))
        }
    }
}
```
With the `localeOverride` parameter, we can generate localized previews in Android Studio.

![Android Studio rendering of AppleCounter composable in English and Czech languages](./preview.avif)

Also, the interactive preview feature of Android Studio allows to click on the buttons and
watch how the localization and plural expressions works perfectly!

import AppPreviewVideo from "./livedemo.avif"

<img lazy width="100%" src={AppPreviewVideo} alt="Android Studio rendering AppleCounter composable live, even with locale switching!"/>

## Summary

In the demo code, you can

* Localize strings
* Better placeholders and conditional substitutions
* Better plural expressions
* Supports annotated strings

However, this is not just limited to strings. You can localize images or even composables
as well. Feel free to adapt this structure to better fit your needs!
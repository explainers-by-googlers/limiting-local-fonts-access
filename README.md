# Limiting Access to Local Fonts

This proposal is an early design sketch by Chrome to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.


## Authors

[Tanushree Bisht](https://github.com/tanushreebisht)

[Zainab Rizvi](https://github.com/zainabaq)


## Participate
[https://github.com/explainers-by-googlers/limiting-local-fonts-access/issues](https://github.com/explainers-by-googlers/limiting-local-fonts-access/issues)

## Introduction
The specific set of fonts available locally on a device is somewhat unique, as different users install different software with different configurations, language support, etc. It is possible to determine which fonts a user has installed locally by measuring the side-effects of font rendering, and these techniques are widely relied-upon as a core component of scripts that aim to re-identify users cross-context.

To limit its utility for re-identification of the browser or device, we propose reducing the number of local fonts that a webpage can load by limiting the returned list of fonts to default system fonts shipped by the operating system. 


## Goals
*   Ensure that the same list of fonts is used for all users on a platform
*   Address the various ways user-installed fonts can be accessed, including CSS `font-family`, CSS `@font-face` and Canvas text rendering
*   Allow users to grant access to all locally-installed fonts to specific origins using the [Local Font Access API](https://developer.chrome.com/docs/capabilities/web-apis/local-fonts) 



## Non-goals
*   This document does not propose solutions for platforms that have varied or customizable font packages, such as Linux, for which it is difficult to build an allowlist of platform fonts due to its wide range of distributions that tailor to varying user preferences
*   The intervention proposed here only impacts the availability of locally-installed fonts; web developers can continue to load fonts from the web as they do presently


## Overview


### Context

#### Motivation

The CSS Working Group has had [ongoing discussions](https://github.com/w3c/csswg-drafts/issues/11775) over limiting the availability of local fonts in such a way that would not break the web and disproportionately impact users who use less common languages. 

[Recent CSSWG discussions](https://github.com/w3c/csswg-drafts/issues/11753) have brought up the idea of prescribing user agents to not expose user-installed fonts on the web as a privacy protecting measure. These measures mirror [Safari’s approach](https://webkit.org/tracking-prevention/) of limiting local font availability by restricting to fonts that are bundled with the operating system by default. 


#### Background

##### How fonts are used to track users

Websites may be able to detect the set of fonts installed on a user’s system, which, in combination with other signals, can be used to re-identify a browser across contexts. 

One of them involves the Canvas API. A particular font can be used to draw text onto a `<canvas>` element. The resulting pixel data can be analyzed via `canvas.measureText()` and compared against known sizes of rendered Canvases from reference fonts. 

Websites can also detect whether a font is installed on a system by rendering text using a font whose existence is being checked, measuring the width of its `<span>`, and comparing it with those of generic fallback fonts. If the widths differ, the font is considered present on the user’s system; otherwise, it is assumed to be missing.


### Implementation

#### Current state

[Blink’s font matching mechanism](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/renderer/platform/fonts/README.md) makes use of system fonts when needed by selecting glyphs based on various characteristics, such as desired fonts, language and character set. It employs [font fallback](https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/renderer/platform/fonts/README.md#font-fallback) and caching mechanisms to accurately render text across various platforms.

Local fonts can be accessed via the following methods:

*   CSS [font-family](https://developer.mozilla.org/en-US/docs/Web/CSS/font-family) property
*   [local() src](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/src#localfont-face-name) to CSS [@font-face rule](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face)
*   [CanvasRenderingContext2D.font](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/font) property
*   [Local Fonts Access API](https://developer.mozilla.org/en-US/docs/Web/API/Local_Font_Access_API)

#### Intervention

Initially, user-installed fonts specified via the CSS [font-family](https://developer.mozilla.org/en-US/docs/Web/CSS/font-family) property, [local() src](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/src#localfont-face-name) for the CSS [@font-face rule](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face), and [CanvasRenderingContext2D.font](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/font) property may be blocked from rendering.

Multiple ways of implementing this approach have been explored. A potential approach is to build and maintain lists of the fonts that various platforms install by default, and to use those lists as sources of truth when determining whether a font is considered user-installed and thus filtered from font selection.

Windows and macOS could be the first platforms that we limit the availability of user-installed fonts on as they have officially published the lists of the fonts they install by default. For each platform and platform version, we will use the flat lists of OS-installed font faces that are published online for a base OS version: 

*   [Windows 10 fonts list](https://learn.microsoft.com/en-us/typography/fonts/windows_10_font_list) to encompass Windows 10/11
*   [macOS 13 Ventura fonts list](https://support.apple.com/en-us/103197) to encompass macOS 13/14/15

The OS-shipped fonts lists sourced from above will be stored in binary itself or loaded at browser startup  time (for easier list updates) during the experimentation phase. In later phases, we may consider delivering the OS-shipped fonts lists to Chrome via more automated and maintainable approaches, as well as storing more granular lists per platform version. 

We will start off by filtering user-installed fonts from being accessed via the `font-family` CSS property, as that is the most common way developers specify what fonts they wish to use to render text on their webpages. We will also collect impact metrics in related codepaths. Filtering will be activated during the font selection process when:

1. The operating system of the user agent is either macOS or Windows
2. The user has not given prior [permission](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Permissions-Policy/local-fonts) to access their local fonts on an origin via the [Local Fonts Access API](https://developer.mozilla.org/en-US/docs/Web/API/Local_Font_Access_API)

#### Analysis 

By running an experiment with the intervention as outlined above, we can get a headstart on quantifying impacts of the filtering of user-installed fonts. We are interested in collecting metrics on the following data points:

*   Font fallback rates
    *   How often we have to select the last resort font when the intervention is enabled.
*   Performance impacts of the intervention
    *   The lookup of whether a font is considered a system font in various codepaths in the fonts codebase will add latency to the font selection mechanism.
*   Less common languages which may have limited font support
    *   These are less common languages and/or languages that have complex or rare scripts that may not often be bundled with the operating systems at install-time, which users may have to manually install on their devices. Local font filtering may disproportionately result in users of such languages having issues with text rendering.

From the analysis on the metrics collected during the experimentation phase, we may look into adjusting the list of fonts we include in the shipped lists to get better character coverage over less frequently used languages on the web.

### Future Improvements

Longer term, we may investigate providing a consistent, limited set of fonts (a "synthetic font environment"), based on existing corpuses. This approach has challenges in terms of distribution, as a full set of fonts would mean a sizable download. We hope insights gained from implementing this current proposal, which limits access to default system fonts, would be valuable in exploring the feasibility and design of such a system.

While we initially aim to ship in Incognito mode, we are evaluating limiting font access to all browsing sessions as a platform-hardening measure. This could potentially involve allowing access to a small, curated list of common fonts beyond system defaults or limiting access to a numerical threshold of locally-installed fonts.

## FAQs

### Limiting to local fonts will break my application, what should I do?

Does your application specifically need access to user-installed local fonts? If so, consider switching to the [Local Font Access API](https://developer.chrome.com/docs/capabilities/web-apis/local-fonts#the_local_font_access_api) which will always be honored with this proposal. If your use case falls outside this, we would like to consider it, so please file an issue.  

### 
How will font lists be maintained? 

We have decided to focus on Windows and MacOS because font lists are readily available and easier to compile ([macOS](https://support.apple.com/en-us/103197), [Windows](https://learn.microsoft.com/en-us/typography/fonts/windows_10_font_list)). For these platforms, we expect to update the list with each major platform release. Ensuring accurate and up-to-date lists will require collaboration with the platform owners in terms of them continuing to share their lists or opening up access to APIs that reliably expose their lists of default OS-shipped fonts.

### Are there plans to introduce this to other platforms?

We are starting with Windows and MacOS because of the ease of availability of font lists.

For Linux, it’s difficult to come up with a common denominator list of OS-shipped fonts because of its wide range of distributions. Linux distributions are customized to fit different target needs and audiences, which can lead to variances in default font packages and flavors. For example, distributions may vary in their default Unicode coverage based on the target audience, and thus tailor their default font packages to specific locales.

Android is also not considered in the initial intervention as Android ships with a limited set of fonts that are controlled by the OS/device manufacturer and does not commonly rely on user-installed fonts. However, fonts via `src: local()` matching may still present identifiable information on Android devices based on variations in available fonts and OEM configurations.

In an ideal world, the user agent is not responsible for maintaining font lists because the OS would provide an API that exposes what fonts are considered OS-defaults.

## Security and privacy considerations

The goal of limiting local fonts is to reduce information available that can be used to re-identify the user across sites, so we expect this proposal to be privacy-positive. Though limiting font availability to OS-installed fonts can expose the user’s platform and potentially the major version, we accept this because user agent headers already make this information available.

## Contributors and Acknowledgements

[Antonio Sartori](https://github.com/antosart)

[Dominik Röttsches](https://github.com/drott)

[Mike West](https://github.com/mikewest)
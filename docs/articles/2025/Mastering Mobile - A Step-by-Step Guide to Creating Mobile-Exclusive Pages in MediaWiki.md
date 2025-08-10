---
layout: default
parent: Contents
date: 2025-07-10
nav_exclude: true
---

## Mastering Mobile: A Step-by-Step Guide to Creating Mobile-Exclusive Pages in MediaWiki
- TOC
{:toc}

In an increasingly mobile-first world, ensuring your MediaWiki site delivers an optimal experience on all devices is crucial. This guide will walk you through the process of creating mobile-exclusive pages and content, allowing you to tailor the user experience for your on-the-go audience. We'll cover everything from the foundational concepts to practical implementation with code snippets and testing tips.

### 1\. Understanding Mobile-Exclusive Content and MediaWiki's Approach

A **mobile-exclusive page** in MediaWiki isn't a separate, standalone page. Instead, it refers to a standard wiki page that is dynamically rendered to display specific content only to users on mobile devices. This is achieved by selectively hiding or showing elements based on the user's device.

MediaWiki's primary tool for this is the **MobileFrontend** extension. When installed, it detects whether a user is accessing the wiki from a desktop or mobile browser and delivers a mobile-optimized view. This is often paired with a mobile-friendly skin, such as **Minerva Neue**, which is designed for smaller screens and touch interactions. MobileFrontend provides the underlying logic and hooks to conditionally display content, making the creation of mobile-exclusive experiences possible.

### 2\. Installing and Enabling the Necessary Tools

To begin, you'll need to have administrator access to your MediaWiki installation to install extensions and modify configuration files.

#### Step 2.1: Install the MobileFrontend Extension

1.  **Download the Extension:** Obtain the MobileFrontend extension files. You can download a snapshot from the [MediaWiki extension repository](https://www.mediawiki.org/wiki/Extension:MobileFrontend) or clone it using Git for the latest version.

2.  **Place it in your `extensions` directory:** Create a folder named `MobileFrontend` within your MediaWiki's `extensions` directory and place the downloaded files there.

3.  **Enable the Extension:** Open your `LocalSettings.php` file, which is located in the root directory of your MediaWiki installation. Add the following line at the end of the file:

    ```php
    wfLoadExtension( 'MobileFrontend' );
    ```

#### Step 2.2: Install and Set a Mobile-Friendly Skin

While MobileFrontend can work with various skins, it's highly recommended to use a skin designed for mobile viewing. Minerva Neue is the standard choice and the one used by Wikipedia's mobile site.

1.  **Download the Minerva Neue Skin:** If it's not already included in your MediaWiki version, download it from the [MediaWiki skin repository](https://www.mediawiki.org/wiki/Skin:Minerva_Neue).

2.  **Place it in your `skins` directory:** Create a folder named `MinervaNeue` in your MediaWiki's `skins` directory and place the files there.

3.  **Enable the Skin and Set it for Mobile:** In your `LocalSettings.php`, add the following lines *after* the `wfLoadExtension( 'MobileFrontend' );` line:

    ```php
    wfLoadSkin( 'MinervaNeue' );
    $wgDefaultMobileSkin = 'minerva';
    ```

    This configuration tells MediaWiki to use the Minerva Neue skin for mobile users.

### 3\. Creating Mobile-Exclusive Content: Methods and Examples

With the necessary tools in place, you can now start tailoring your content for mobile devices. Here are several methods, ranging from simple CSS classes to more advanced extensions.

#### Method 3.1: Using CSS Classes (`mobileonly` and `nomobile`)

MobileFrontend introduces two crucial CSS classes that you can use directly in your wikitext to control content visibility.

  * `.mobileonly`: Content with this class will *only* be visible on mobile devices.
  * `.nomobile`: Content with this class will be *hidden* on mobile devices.

To use them, wrap your content in a `<div>` or `<span>` tag with the appropriate class.

**Example:**

```wikitext
This text is visible on all devices.

<div class="mobileonly">
This content will only appear on mobile devices. You could put a mobile-specific announcement or a simplified table here.
</div>

<div class="nomobile">
This content will only appear on desktop devices. This might include a wide, complex table or a large image gallery that doesn't render well on smaller screens.
</div>

You can also use these for inline elements: This is some regular text, but <span class="nomobile">this part is for desktops only</span> and <span class="mobileonly">this part is for mobile only</span>.
```

To ensure these classes function correctly, you may need to add the following styles to your wiki's CSS pages.

1.  Navigate to `MediaWiki:Common.css` on your wiki and add:

    ```css
    /* Hide mobile-only content on desktop */
    .mobileonly {
        display: none;
    }
    ```

2.  Navigate to `MediaWiki:Mobile.css` on your wiki (create the page if it doesn't exist) and add:

    ```css
    /* Hide desktop-only content on mobile */
    .nomobile {
        display: none;
    }
    ```

#### Method 3.2: The MobileDetect Extension for Simpler Syntax

For a more user-friendly syntax, you can install the **MobileDetect** extension.

1.  **Install the Extension:** Follow the same installation procedure as for MobileFrontend.
2.  **Enable in `LocalSettings.php`:**
    ```php
    wfLoadExtension( 'MobileDetect' );
    ```

This extension provides parser tags that are easier for non-technical users to remember and use.

  * `<mobileonly>`...`</mobileonly>`:  Shows content only on mobile.
  * `<nomobile>`...`</nomobile>`: Hides content on mobile.

**Example:**

```wikitext
This is visible on all devices.

<mobileonly>
This is a much cleaner way to show content only on mobile!
</mobileonly>

<nomobile>
And this is a simpler syntax for desktop-only content.
</nomobile>
```

#### Method 3.3: Creating a Mobile-Exclusive Main Page

You can create a completely different layout for your wiki's main page for mobile users.

1.  **Enable in `LocalSettings.php`:** Add the following line to your `LocalSettings.php`:
    ```php
    $wgMFSpecialCaseMainPage = true;
    ```
2.  **Create Mobile Main Page Content:** On your wiki's main page (e.g., "Main Page"), you can now use the `id="mp-..."` selector to define sections that will be visible on mobile. All other content outside of these designated sections will be hidden on the mobile view of the main page.

**Example on "Main Page":**

```wikitext
<div id="mp-top">
  '''Welcome to Our Wiki!''' This is the mobile-friendly introduction.
</div>

<div id="mp-middle">
  == Today's Featured Article ==
  A brief, mobile-friendly excerpt of the featured article.
</div>

<div class="nomobile">
This entire section with complex layouts and multiple columns will not be shown on the mobile main page.
</div>
```

### 4\. Custom CSS and JavaScript for Advanced Control

For more granular control over the mobile experience, you can use custom CSS and JavaScript.

#### Custom CSS in `MediaWiki:Mobile.css`

You can add any CSS rules to `MediaWiki:Mobile.css` to style the mobile view. For example, you might want to adjust font sizes, change colors, or modify the layout of certain elements specifically for mobile.

**Example in `MediaWiki:Mobile.css`:**

```css
/* Increase the font size of the main content on mobile for better readability */
.mw-body-content {
    font-size: 1.1em;
}

/* Hide a specific, non-essential sidebar element on mobile */
#p-navigation {
    display: none;
}
```

#### Custom JavaScript in `MediaWiki:Mobile.js`

For dynamic changes, you can use JavaScript in `MediaWiki:Mobile.js`. This script is loaded only for mobile users.

**Example in `MediaWiki:Mobile.js`:**

```javascript
// A simple example to add a "Go to Top" button on mobile
( function () {
    var toTopButton = document.createElement( 'button' );
    toTopButton.innerHTML = '↑ Top';
    toTopButton.style.position = 'fixed';
    toTopButton.style.bottom = '20px';
    toTopButton.style.right = '20px';
    toTopButton.style.display = 'none';
    toTopButton.style.zIndex = '100';
    document.body.appendChild( toTopButton );

    window.addEventListener( 'scroll', function () {
        if ( window.scrollY > 300 ) {
            toTopButton.style.display = 'block';
        } else {
            toTopButton.style.display = 'none';
        }
    } );

    toTopButton.addEventListener( 'click', function () {
        window.scrollTo( { top: 0, behavior: 'smooth' } );
    } );
}() );
```

### 5\. Optional: Tips for Testing and Previewing

It's essential to test how your mobile-exclusive content appears. Here are a few ways to do this:

  * **URL Parameter:** The simplest way to preview the mobile view on a desktop browser is to add `?useformat=mobile` to the end of any page's URL. For example: `https://yourwiki.com/index.php/My_Page?useformat=mobile`. To switch back, you can use `?useformat=desktop`.

  * **Browser Developer Tools:** All modern web browsers (Chrome, Firefox, Edge, Safari) have built-in developer tools that can simulate various mobile devices.

      * **How to use (general steps):**
        1.  Right-click on your page and select "Inspect" or "Inspect Element."
        2.  Look for an icon that depicts a mobile phone or tablet (often called "Toggle Device Toolbar" or similar).
        3.  Click this icon to enter mobile simulation mode. You can then choose from a list of popular devices to see how your page renders on different screen sizes and resolutions.

By following these steps, you can effectively create a more engaging and user-friendly experience for your mobile audience, ensuring that your MediaWiki content is accessible and well-presented, no matter how it's being viewed.
---
title: How to find the menu items in WordPress
date: 2021-08-14 00:10:57
tags:
  - WordPress
---

## Overview

This article introduces how to get the menu object and menu path corresponding to the current page, how to get the menu object's sub menus, how to get the menu object's parent object, etc. in WordPress. Even if you don't need to define your own menu presentation, you can get inspiration from the implementation of these functions to be able to do more flexible customization in other places.

<!-- more -->

## Why you need these helper function

If you want to be able to use the menu in WordPress more flexibly, you may want to decide all the layout code of the menu by yourself.

## The Helper function code

```php

<?php

/**
 * Find sub menu items of a parent menu item.
 * if $menu_item_parent parameter is 0, return all top menu items
 */
function findMenuItems($location, $menu_item_parent)
{
    $items = get_nav_menu_items_by_location($location);
    $rtn = array();
    foreach ($items as $item) {
        if ($item->menu_item_parent == $menu_item_parent) {
            $rtn[] = $item;
        }
    }
    return $rtn;
}

/**
 * Find sub menus of a parent menu item
 */
function findMenuItemsByParentID($menu_item_parent)
{
    $locations = get_nav_menu_locations();
    foreach ($locations as $location => $v) {
        $rtn = findMenuItems($location, $menu_item_parent);
        if (!empty($rtn) && count($rtn) > 0) {
            return $rtn;
        }
    }
    return array();
}


/**
 * Get the parent menu item by the specific menu item
 */
function findParentMenuItem($menuItem)
{
    if (empty($menuItem)) return 0;
    $pid = $menuItem->menu_item_parent;
    $parent = findMenuItemById($pid);
    return $parent;
}

/**
 * You can define a menu tree in your backend of WordPress.
 *
 * When you visit a page, according the URI of the page, find
 * the matching menu item and all its ancestor menu items.
 * [0]: The top menu item
 * [1]: The menu item in level 1 of the menu tree
 * [2]: The menu item in level 2 of the menu tree
 * ...
 *
 * for example, if you have a menu tree :
 * A
 *   A1
 *   A2
 *     A21
 *     A22
 * B
 *   B1
 *      B11
 *   B2
 *
 * And the current URI is matching the "B11" menu,you will get:
 * [0] = B
 * [1] = B1
 * [2] = B11
 *
 * So, You can set the B, B1, B11 to selected state on your website!
 */
function getCurrentMenuStack()
{
    $rtn = array();

    $subMenu = findCurrentMenuItem();
    if (!empty($subMenu)) {
        $rtn[] = $subMenu;
    }
    while (!empty($subMenu->menu_item_parent)) {
        $subMenu = findParentMenuItem($subMenu);
        if (!empty($subMenu)) {
            $rtn[] = $subMenu;
        }
    }
    return array_reverse($rtn);
}

/**
 * Get current Menu Item Object
 * If current request URI is matching a menu item,return the menu item.
 */
function findCurrentMenuItem()
{
    $uri = $_SERVER["REQUEST_URI"];
    if (home_url("/") == get_permalink()) {
        return "";
    }
    return findMenuItem($uri);
}

/**
 * You should not call this function directly
 */
function findMenuItemById($menu_object_id)
{
    if (empty($menu_object_id)) {
        return 0;
    }
    $locations = get_nav_menu_locations();
    foreach ($locations as $location => $v) {
        $parents = findMenuItems($location, 0);
        foreach ($parents as $parent) {
            $rtn = hasEqualsMenuItem($location, $parent, $menu_object_id);
            if (!empty($rtn)) {
                return $rtn;
            }
        }
    }

    throw new \Exception("Not found any MenuItem with menu ID [$menu_object_id]");
}

/**
 * You should not call this function directly
 */
function hasEqualsMenuItem($location, $menuItem, $menu_object_id)
{
    if ($menuItem->object_id == $menu_object_id) {
        return $menuItem;
    }

    $children = findMenuItems($location, $menuItem->object_id);
    foreach ($children as $child) {
        $rtn = hasEqualsMenuItem($location, $child, $menu_object_id);
        if (!empty($rtn)) {
            return $rtn;
        }
    }
    return 0;
}


/**
 * You should not call this function directly
 */
function findMenuItem($uri)
{
    $locations = get_nav_menu_locations();
    $menuItemStack = array();

    foreach ($locations as $location => $v) {
        $parents = findMenuItems($location, 0);

        foreach ($parents as $parent) {
            privateFindMenuStack($location, $parent, $uri, $menuItemStack, 0);
        }
    }
    if (count($menuItemStack) == 0) {
        return 0;
    }

    return end($menuItemStack)["menu_item"];
}

/**
 * You should not call this function directly
 */
function privateFindMenuStack($location, $menuItem, $uri, &$menuItemStack, $level)
{
    $menuItemUrl = $menuItem->url;
    if (
        substr($uri, 0, strlen($menuItemUrl)) === $menuItemUrl
        || substr($menuItemUrl, 0, strlen($uri)) === $uri
    ) {
        $menuItemStack[] = array("menu_item" => $menuItem, "level" => $level);
    }

    $children = findMenuItems($location, $menuItem->object_id);
    if (count($children) != 0) {
        foreach ($children as $child) {
            privateFindMenuStack($location, $child, $uri, $menuItemStack, $level + 1);
        }
    }
}

/**
 * You should not call this function directly
 */
function get_nav_menu_items_by_location($location, $args = [])
{

    // Get all locations
    $locations = get_nav_menu_locations();

    // Get object id by location
    if (!isset($locations[$location])) {
        return array();
    }
    $object = wp_get_nav_menu_object($locations[$location]);

    if (empty($object)) {
        return array();
    }

    // Get menu items by menu name
    $menu_items = wp_get_nav_menu_items($object->name, $args);

    // Return menu post objects
    return $menu_items;
}


```

## Example

```php
<?php

$currentMenuStack = getCurrentMenuStack();

/**
 * Display the current crumb navigation
 */
foreach ($currentMenuStack as $menuItem) {
    echo "&gt;" . $menuItem->title;
}

/**
 * Display the top menus
 */
$topPrimaryMenus = findMenuItems("primary", 0);
echo "<ul>";
foreach ($topPrimaryMenus as $topMenuItem) {
    $selected = "";
    if ($topMenuItem->object_id == $currentMenuStack[0]->object_id) {
        $selected = "selected";
    }
    echo "<li class=\"" . $selected . "\"> <a href=\"" . $topMenuItem->url . "\">" . $topMenuItem->title . "</a></li>";
}
echo "</ul>";

```

Enjoy it!

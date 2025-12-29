---
inclusion: always
---

# Odoo Design System Rules

## Overview

This document provides design system guidelines for integrating Figma designs with the Odoo 19 codebase. Odoo uses a server-side rendering approach with QWeb templates and Python backend logic.

## Design System Structure

### Token Definitions

Odoo uses CSS variables and SCSS for styling. Design tokens are typically defined in:
- **Location**: `odoo/addons/*/static/src/css/` and `odoo/addons/*/static/src/scss/`
- **Format**: SCSS variables and CSS custom properties
- **Token Types**:
  - Colors: Primary, secondary, success, warning, danger, info
  - Typography: Font families, sizes, weights, line heights
  - Spacing: Margin and padding scales (4px, 8px, 12px, 16px, etc.)
  - Shadows: Elevation levels for depth
  - Border radius: Rounded corner scales

### Component Library

Odoo components are organized as:
- **Backend Components**: Python classes in `odoo/models/` and `odoo/fields/`
- **Frontend Components**: JavaScript/XML templates in `odoo/addons/*/static/src/js/`
- **QWeb Templates**: XML-based templating in `odoo/addons/*/views/`
- **Web Components**: Reusable UI elements in `odoo/addons/web/static/src/js/`

### Frameworks & Libraries

- **Backend**: Python 3.x with Odoo ORM
- **Frontend**: JavaScript (ES6+), jQuery (legacy), Owl framework (modern)
- **Styling**: SCSS/CSS with Bootstrap 4/5 integration
- **Build System**: Odoo's asset bundling system (manifest.py)
- **Templating**: QWeb (Odoo's XML-based template engine)

### Asset Management

- **Location**: `odoo/addons/*/static/src/`
- **Structure**:
  - `img/` - Images and graphics
  - `icons/` - Icon assets
  - `css/` - Stylesheets
  - `js/` - JavaScript files
  - `xml/` - QWeb templates
- **Optimization**: Odoo automatically minifies and bundles assets in production

### Icon System

- **Format**: SVG and Font Awesome icons
- **Location**: `odoo/addons/web/static/src/img/icons/`
- **Usage**: Inline SVG or icon font classes
- **Naming Convention**: `icon-{name}` or Font Awesome class names

### Styling Approach

- **Methodology**: SCSS with BEM-like naming conventions
- **Global Styles**: `odoo/addons/web/static/src/css/`
- **Responsive Design**: Mobile-first approach with Bootstrap breakpoints
- **CSS Variables**: Used for theming and dynamic styling
- **Scoping**: Component-level styles in addon-specific CSS files

### Project Structure

```
odoo/
├── addons/              # Core and community addons
│   ├── web/            # Web framework addon
│   ├── account/        # Accounting addon
│   └── ...
├── api/                # API modules
├── fields/             # Field definitions
├── models/             # ORM models
├── orm/                # ORM core
└── tools/              # Utility functions
```

## Integration Guidelines for Figma

### When Converting Figma Designs to Odoo

1. **Use QWeb Templates**: Convert Figma layouts to QWeb XML templates
2. **Leverage Bootstrap**: Odoo uses Bootstrap, so use Bootstrap classes for layouts
3. **Apply SCSS Variables**: Use Odoo's design tokens instead of hardcoded values
4. **Component Reuse**: Check existing Odoo components before creating new ones
5. **Responsive Design**: Ensure designs work on desktop, tablet, and mobile

### File Naming Conventions

- **Templates**: `{feature_name}_templates.xml`
- **Stylesheets**: `{feature_name}.scss`
- **JavaScript**: `{feature_name}.js`
- **Python Models**: `{feature_name}.py`

### Code Organization

- Keep templates in `views/` directory
- Keep styles in `static/src/css/` or `static/src/scss/`
- Keep JavaScript in `static/src/js/`
- Keep Python logic in addon root or `models/` subdirectory

## Design Tokens

### Colors

```scss
$primary: #007bff;
$secondary: #6c757d;
$success: #28a745;
$warning: #ffc107;
$danger: #dc3545;
$info: #17a2b8;
$light: #f8f9fa;
$dark: #343a40;
```

### Typography

```scss
$font-family-base: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
$font-size-base: 1rem;
$font-size-lg: 1.25rem;
$font-size-sm: 0.875rem;
$line-height-base: 1.5;
```

### Spacing

```scss
$spacer: 1rem;
$spacers: (
  0: 0,
  1: $spacer * 0.25,
  2: $spacer * 0.5,
  3: $spacer,
  4: $spacer * 1.5,
  5: $spacer * 3
);
```

## Best Practices

1. **Consistency**: Use design tokens consistently across all components
2. **Accessibility**: Ensure sufficient color contrast and keyboard navigation
3. **Performance**: Minimize CSS and optimize images
4. **Maintainability**: Keep styles organized and documented
5. **Testing**: Test designs on multiple browsers and devices
6. **Documentation**: Document custom components and their usage

## Resources

- [Odoo Documentation](https://www.odoo.com/documentation/)
- [QWeb Template Engine](https://www.odoo.com/documentation/17.0/developer/reference/frontend/qweb.html)
- [Bootstrap Documentation](https://getbootstrap.com/docs/)
- [SCSS Documentation](https://sass-lang.com/documentation)

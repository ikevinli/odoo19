# Design Document: Custom Packaging E-Commerce Module

## Overview

The custom_packaging module extends Odoo 19's core e-commerce functionality to provide a specialized platform for selling customizable, sustainable packaging products. The system integrates with Odoo's product, website_sale, and sale modules while adding custom design tools, dynamic pricing, and proof approval workflows.

### Key Design Principles

1. **Extend, Don't Replace**: Leverage existing Odoo modules (product, website_sale, sale) through inheritance
2. **Client-Side Performance**: Use Fabric.js for responsive design manipulation without server round-trips
3. **Progressive Enhancement**: Provide fallback options when JavaScript features are unavailable
4. **Sustainability First**: Emphasize eco-friendly materials and certifications throughout the user experience
5. **Mobile-First**: Design responsive interfaces that work seamlessly on all devices

## Architecture

### Module Structure

```
custom_packaging/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── product_template.py          # Extends product.template
│   ├── product_product.py           # Extends product.product
│   ├── product_packaging_material.py # New model for materials
│   ├── product_packaging_template.py # New model for design templates
│   ├── sale_order_line.py           # Extends sale.order.line
│   ├── packaging_design.py          # New model for customer designs
│   └── packaging_certification.py   # New model for sustainability certs
├── controllers/
│   ├── __init__.py
│   ├── main.py                      # Main website controller
│   └── design_tool.py               # Design tool API endpoints
├── views/
│   ├── product_template_views.xml   # Backend product forms
│   ├── product_website_templates.xml # Frontend product pages
│   ├── design_tool_templates.xml    # Design tool interface
│   ├── cart_templates.xml           # Shopping cart customizations
│   ├── checkout_templates.xml       # Checkout flow
│   └── portal_templates.xml         # Customer portal for proofs
├── static/
│   ├── src/
│   │   ├── js/
│   │   │   ├── design_tool.js       # Fabric.js design tool
│   │   │   ├── pricing_calculator.js # Dynamic pricing
│   │   │   ├── cart_customization.js # Cart interactions
│   │   │   └── file_uploader.js     # File upload handler
│   │   ├── scss/
│   │   │   ├── design_tool.scss     # Design tool styles
│   │   │   ├── product_page.scss    # Product page styles
│   │   │   └── mobile.scss          # Mobile-specific styles
│   │   └── xml/
│   │       └── qweb_templates.xml   # QWeb templates for JS
│   └── lib/
│       └── fabric.min.js            # Fabric.js library
├── wizard/
│   ├── __init__.py
│   └── design_proof_wizard.py       # Proof generation wizard
├── report/
│   ├── __init__.py
│   ├── design_proof_report.xml      # PDF proof template
│   └── design_proof_report.py       # Proof generation logic
├── security/
│   ├── ir.model.access.csv          # Access control
│   └── security.xml                 # Record rules
└── data/
    ├── product_data.xml             # Default materials, certs
    └── email_templates.xml          # Email templates
```

### Technology Stack

- **Backend**: Python 3.10+, Odoo 19 ORM
- **Frontend**: JavaScript ES6+, Fabric.js 5.x, jQuery (Odoo legacy)
- **Styling**: SCSS with Bootstrap 5 (Odoo's framework)
- **Templating**: QWeb (XML-based)
- **Database**: PostgreSQL 14+
- **File Storage**: Odoo's attachment system (ir.attachment)

## Components and Interfaces

### 1. Extended Product Models

#### product.template Extension

```python
class ProductTemplate(models.Model):
    _inherit = 'product.template'
    
    # Packaging-specific fields
    is_custom_packaging = fields.Boolean('Is Custom Packaging Product')
    packaging_category = fields.Selection([
        ('tissue', 'Tissue Paper'),
        ('mailer', 'Mailers'),
        ('box', 'Boxes'),
        ('sticker', 'Stickers'),
        ('tape', 'Tape'),
        ('tote', 'Tote Bags'),
    ], string='Packaging Category')
    
    # MOQ Configuration (product-specific defaults)
    minimum_order_qty = fields.Integer('Minimum Order Quantity', default=100)
    moq_help_text = fields.Char('MOQ Help Text', compute='_compute_moq_help_text')
    
    # Material Options
    material_ids = fields.Many2many('product.packaging.material', string='Available Materials')
    default_material_id = fields.Many2one('product.packaging.material', string='Default Material')
    
    # Print Options
    max_colors = fields.Integer('Maximum Colors', default=4)
    print_type_ids = fields.Many2many('product.print.type', string='Available Print Types')
    supports_double_sided = fields.Boolean('Supports Double-Sided Printing')
    price_per_additional_color = fields.Float('Price Per Additional Color', default=0.05)
    double_sided_price_add = fields.Float('Double-Sided Additional Cost', default=0.10)
    
    # Design Configuration
    design_template_ids = fields.One2many('product.packaging.template', 'product_tmpl_id', string='Design Templates')
    mockup_image = fields.Binary('Product Mockup Image')
    mockup_back_image = fields.Binary('Product Mockup Back Image')  # For double-sided
    design_area_width = fields.Float('Design Area Width (mm)')
    design_area_height = fields.Float('Design Area Height (mm)')
    supports_pattern_repeat = fields.Boolean('Supports Pattern Repeat')
    
    # Product-specific features
    supports_die_cut = fields.Boolean('Supports Die-Cut Shapes')  # For stickers/tape
    has_inside_print = fields.Boolean('Has Inside Print Area')  # For boxes
    
    # Sustainability
    certification_ids = fields.Many2many('packaging.certification', string='Certifications')
    sustainability_description = fields.Html('Sustainability Description')
    carbon_footprint_reduction = fields.Float('Carbon Footprint Reduction %', help='Percentage reduction vs standard materials')
    uses_soy_ink = fields.Boolean('Uses Soy-Based Ink')
    
    # Pricing Tiers
    quantity_tier_ids = fields.One2many('product.quantity.tier', 'product_tmpl_id', string='Quantity Price Tiers')
    
    @api.depends('packaging_category')
    def _compute_moq_help_text(self):
        """Set MOQ defaults based on product category"""
        moq_defaults = {
            'tissue': 250,
            'mailer': 100,
            'box': 100,
            'sticker': 250,
            'tape': 250,
            'tote': 50,
        }
        for product in self:
            if product.packaging_category:
                default_moq = moq_defaults.get(product.packaging_category, 100)
                if not product.minimum_order_qty or product.minimum_order_qty == 100:
                    product.minimum_order_qty = default_moq
                product.moq_help_text = f'Industry standard MOQ: {default_moq} units'
            else:
                product.moq_help_text = ''
    
    @api.onchange('packaging_category')
    def _onchange_packaging_category(self):
        """Set category-specific defaults"""
        if self.packaging_category == 'tissue':
            self.supports_pattern_repeat = True
            self.minimum_order_qty = 250
        elif self.packaging_category == 'mailer':
            self.supports_double_sided = True
            self.minimum_order_qty = 100
        elif self.packaging_category == 'box':
            self.has_inside_print = True
            self.supports_double_sided = True
            self.minimum_order_qty = 100
        elif self.packaging_category in ['sticker', 'tape']:
            self.supports_die_cut = True
            self.minimum_order_qty = 250
        elif self.packaging_category == 'tote':
            self.minimum_order_qty = 50
```

#### product.packaging.material Model

```python
class ProductPackagingMaterial(models.Model):
    _name = 'product.packaging.material'
    _description = 'Packaging Material Option'
    
    name = fields.Char('Material Name', required=True)
    code = fields.Char('Material Code')
    description = fields.Text('Description')
    is_sustainable = fields.Boolean('Sustainable Material')
    price_multiplier = fields.Float('Price Multiplier', default=1.0, help='Multiplier applied to base price')
    price_add_per_unit = fields.Float('Additional Cost Per Unit', default=0.0, help='Fixed cost added per unit')
    certification_ids = fields.Many2many('packaging.certification', string='Certifications')
    icon = fields.Binary('Material Icon')
    
    # Material-specific properties
    material_type = fields.Selection([
        ('recycled_paper', 'Recycled Paper'),
        ('standard_paper', 'Standard Paper'),
        ('compostable', 'Compostable'),
        ('kraft', 'Kraft Paper'),
        ('fsc_cardboard', 'FSC Recycled Cardboard'),
        ('organic_cotton', 'Organic Cotton'),
    ], string='Material Type')
    
    carbon_reduction_pct = fields.Float('Carbon Reduction %', help='Percentage reduction vs standard')
    
    @api.depends('name', 'price_add_per_unit')
    def _compute_display_name(self):
        for material in self:
            if material.price_add_per_unit > 0:
                material.display_name = f'{material.name} (+${material.price_add_per_unit:.2f}/unit)'
            else:
                material.display_name = material.name

#### product.quantity.tier Model

```python
class ProductQuantityTier(models.Model):
    _name = 'product.quantity.tier'
    _description = 'Quantity-Based Pricing Tier'
    _order = 'min_quantity'
    
    product_tmpl_id = fields.Many2one('product.template', string='Product', required=True, ondelete='cascade')
    min_quantity = fields.Integer('Minimum Quantity', required=True)
    unit_price = fields.Float('Unit Price', required=True)
    discount_pct = fields.Float('Discount %', compute='_compute_discount_pct', store=True)
    
    @api.depends('unit_price', 'product_tmpl_id.list_price')
    def _compute_discount_pct(self):
        for tier in self:
            if tier.product_tmpl_id.list_price > 0:
                tier.discount_pct = ((tier.product_tmpl_id.list_price - tier.unit_price) / 
                                    tier.product_tmpl_id.list_price * 100)
            else:
                tier.discount_pct = 0.0
    
    @api.constrains('min_quantity')
    def _check_min_quantity(self):
        for tier in self:
            if tier.min_quantity < tier.product_tmpl_id.minimum_order_qty:
                raise ValidationError(_('Tier minimum quantity cannot be less than product MOQ'))

#### product.print.type Model

```python
class ProductPrintType(models.Model):
    _name = 'product.print.type'
    _description = 'Print Type Option'
    
    name = fields.Char('Print Type Name', required=True)
    code = fields.Char('Print Type Code')
    description = fields.Text('Description')
    additional_cost = fields.Float('Additional Cost Per Unit', default=0.0)
    max_colors = fields.Integer('Maximum Colors', help='0 = unlimited (full color)')
    is_soy_based = fields.Boolean('Soy-Based Ink')
    
    print_method = fields.Selection([
        ('digital', 'Digital Printing'),
        ('offset', 'Offset Printing'),
        ('screen', 'Screen Printing'),
        ('flexo', 'Flexographic Printing'),
    ], string='Print Method')
```

#### packaging.certification Model

```python
class PackagingCertification(models.Model):
    _name = 'packaging.certification'
    _description = 'Sustainability Certification'
    
    name = fields.Char('Certification Name', required=True)
    code = fields.Char('Certification Code')
    description = fields.Text('Description')
    badge_image = fields.Binary('Badge Image')
    verification_url = fields.Char('Verification URL')
```

### 2. Design Storage Model

#### packaging.design Model

```python
class PackagingDesign(models.Model):
    _name = 'packaging.design'
    _description = 'Customer Packaging Design'
    
    name = fields.Char('Design Name', compute='_compute_name', store=True)
    sale_order_line_id = fields.Many2one('sale.order.line', string='Order Line')
    product_id = fields.Many2one('product.product', string='Product', required=True)
    
    # Design Data
    design_file_id = fields.Many2one('ir.attachment', string='Uploaded Design File')
    design_file_back_id = fields.Many2one('ir.attachment', string='Back Design File')  # For double-sided
    canvas_json = fields.Text('Canvas JSON Data')  # Fabric.js canvas state
    canvas_json_back = fields.Text('Back Canvas JSON Data')  # For double-sided
    thumbnail = fields.Binary('Design Thumbnail', compute='_compute_thumbnail', store=True)
    
    # Configuration
    material_id = fields.Many2one('product.packaging.material', string='Selected Material')
    color_count = fields.Integer('Number of Colors', default=1)
    print_type_id = fields.Many2one('product.print.type', string='Print Type')
    quantity = fields.Integer('Quantity')
    is_double_sided = fields.Boolean('Double-Sided Printing')
    uses_pattern_repeat = fields.Boolean('Uses Pattern Repeat')
    
    # Die-cut options (for stickers/tape)
    die_cut_shape = fields.Selection([
        ('circle', 'Circle'),
        ('square', 'Square'),
        ('rectangle', 'Rectangle'),
        ('custom', 'Custom Shape'),
    ], string='Die-Cut Shape')
    
    # Pricing
    unit_price = fields.Float('Unit Price', compute='_compute_price', store=True)
    total_price = fields.Float('Total Price', compute='_compute_price', store=True)
    base_price = fields.Float('Base Price')
    material_cost = fields.Float('Material Cost')
    color_cost = fields.Float('Color Cost')
    print_cost = fields.Float('Print Type Cost')
    
    # Proof Management
    proof_status = fields.Selection([
        ('draft', 'Draft'),
        ('pending', 'Pending Approval'),
        ('approved', 'Approved'),
        ('rejected', 'Rejected'),
    ], default='draft')
    proof_file_id = fields.Many2one('ir.attachment', string='Proof PDF')
    proof_notes = fields.Text('Proof Notes')
    proof_approved_date = fields.Datetime('Proof Approved Date')
    
    @api.depends('product_id', 'material_id', 'color_count', 'print_type_id', 'quantity', 'is_double_sided')
    def _compute_price(self):
        """Calculate dynamic pricing based on configuration"""
        for design in self:
            if not design.product_id or not design.quantity:
                design.unit_price = 0.0
                design.total_price = 0.0
                continue
            
            product = design.product_id.product_tmpl_id
            
            # Start with base price
            base_price = product.list_price
            design.base_price = base_price
            
            # Apply material pricing
            material_cost = 0.0
            if design.material_id:
                material_cost = (base_price * (design.material_id.price_multiplier - 1.0) + 
                               design.material_id.price_add_per_unit)
            design.material_cost = material_cost
            
            # Apply color pricing (first color included in base)
            color_cost = 0.0
            if design.color_count > 1:
                color_cost = (design.color_count - 1) * product.price_per_additional_color
            design.color_cost = color_cost
            
            # Apply print type cost
            print_cost = 0.0
            if design.print_type_id:
                print_cost = design.print_type_id.additional_cost
            design.print_cost = print_cost
            
            # Apply double-sided cost
            double_sided_cost = 0.0
            if design.is_double_sided:
                double_sided_cost = product.double_sided_price_add
            
            # Calculate unit price before quantity discount
            unit_price = base_price + material_cost + color_cost + print_cost + double_sided_cost
            
            # Apply quantity tier pricing
            tier = product.quantity_tier_ids.filtered(
                lambda t: design.quantity >= t.min_quantity
            ).sorted(key=lambda t: t.min_quantity, reverse=True)
            
            if tier:
                unit_price = tier[0].unit_price + material_cost + color_cost + print_cost + double_sided_cost
            
            design.unit_price = unit_price
            design.total_price = unit_price * design.quantity
    
    @api.depends('design_file_id')
    def _compute_thumbnail(self):
        """Generate thumbnail from design file"""
        for design in self:
            if design.design_file_id and design.design_file_id.datas:
                try:
                    image_data = base64.b64decode(design.design_file_id.datas)
                    image = Image.open(io.BytesIO(image_data))
                    image.thumbnail((200, 200), Image.Resampling.LANCZOS)
                    
                    output = io.BytesIO()
                    image.save(output, format='PNG')
                    design.thumbnail = base64.b64encode(output.getvalue())
                except Exception as e:
                    _logger.error('Failed to generate thumbnail: %s', str(e))
                    design.thumbnail = False
            else:
                design.thumbnail = False
```

### 3. Sale Order Line Extension

```python
class SaleOrderLine(models.Model):
    _inherit = 'sale.order.line'
    
    packaging_design_id = fields.Many2one('packaging.design', string='Packaging Design')
    is_custom_packaging = fields.Boolean(related='product_id.product_tmpl_id.is_custom_packaging')
    
    @api.depends('packaging_design_id')
    def _compute_price_unit(self):
        """Override to use custom pricing from design"""
        for line in self:
            if line.packaging_design_id:
                line.price_unit = line.packaging_design_id.unit_price
            else:
                super(SaleOrderLine, line)._compute_price_unit()
```

### 4. Design Tool Controller

```python
class DesignToolController(http.Controller):
    
    @http.route('/shop/product/<model("product.template"):product>/customize', 
                type='http', auth='public', website=True)
    def customize_product(self, product, **kwargs):
        """Render the design tool interface"""
        values = {
            'product': product,
            'materials': product.material_ids,
            'templates': product.design_template_ids,
            'max_colors': product.max_colors,
            'print_types': product.print_type_ids,
        }
        return request.render('custom_packaging.design_tool_page', values)
    
    @http.route('/shop/design/upload', type='json', auth='public', methods=['POST'])
    def upload_design_file(self, file_data, filename, **kwargs):
        """Handle design file upload"""
        # Validate file
        if not self._validate_file(file_data, filename):
            return {'error': 'Invalid file format or size'}
        
        # Create attachment
        attachment = request.env['ir.attachment'].sudo().create({
            'name': filename,
            'datas': file_data,
            'res_model': 'packaging.design',
            'public': False,
        })
        
        return {
            'attachment_id': attachment.id,
            'url': f'/web/content/{attachment.id}',
        }
    
    @http.route('/shop/design/calculate_price', type='json', auth='public')
    def calculate_price(self, product_id, material_id, color_count, print_type_id, quantity, is_double_sided=False, **kwargs):
        """Calculate dynamic price based on configuration"""
        product = request.env['product.product'].browse(product_id)
        material = request.env['product.packaging.material'].browse(material_id)
        print_type = request.env['product.print.type'].browse(print_type_id)
        
        product_tmpl = product.product_tmpl_id
        
        # Base price from product
        base_price = product.list_price
        
        # Apply material pricing
        material_cost = 0.0
        if material:
            material_cost = (base_price * (material.price_multiplier - 1.0) + 
                           material.price_add_per_unit)
        
        # Apply color pricing (first color included)
        color_cost = 0.0
        if color_count > 1:
            color_cost = (color_count - 1) * product_tmpl.price_per_additional_color
        
        # Apply print type cost
        print_cost = 0.0
        if print_type:
            print_cost = print_type.additional_cost
        
        # Apply double-sided cost
        double_sided_cost = 0.0
        if is_double_sided and product_tmpl.supports_double_sided:
            double_sided_cost = product_tmpl.double_sided_price_add
        
        # Calculate unit price before quantity discount
        unit_price = base_price + material_cost + color_cost + print_cost + double_sided_cost
        
        # Apply quantity tier pricing
        tier = product_tmpl.quantity_tier_ids.filtered(
            lambda t: quantity >= t.min_quantity
        ).sorted(key=lambda t: t.min_quantity, reverse=True)
        
        if tier:
            # Use tier base price but keep material/color/print costs
            unit_price = tier[0].unit_price + material_cost + color_cost + print_cost + double_sided_cost
        
        # Get all tiers for display
        tiers = []
        for t in product_tmpl.quantity_tier_ids.sorted(key=lambda x: x.min_quantity):
            tier_price = t.unit_price + material_cost + color_cost + print_cost + double_sided_cost
            tiers.append({
                'min_quantity': t.min_quantity,
                'unit_price': tier_price,
                'total_price': tier_price * t.min_quantity,
                'discount_pct': t.discount_pct,
            })
        
        return {
            'unit_price': unit_price,
            'total_price': unit_price * quantity,
            'currency': request.env.company.currency_id.symbol,
            'breakdown': {
                'base_price': base_price,
                'material_cost': material_cost,
                'color_cost': color_cost,
                'print_cost': print_cost,
                'double_sided_cost': double_sided_cost,
            },
            'tiers': tiers,
        }
    
    @http.route('/shop/design/save', type='json', auth='public')
    def save_design(self, product_id, design_data, **kwargs):
        """Save design configuration"""
        design = request.env['packaging.design'].sudo().create({
            'product_id': product_id,
            'design_file_id': design_data.get('attachment_id'),
            'canvas_json': design_data.get('canvas_json'),
            'material_id': design_data.get('material_id'),
            'color_count': design_data.get('color_count'),
            'print_type_id': design_data.get('print_type_id'),
            'quantity': design_data.get('quantity'),
        })
        
        return {'design_id': design.id}
    
    @http.route('/shop/design/<int:design_id>/add_to_cart', type='json', auth='public')
    def add_design_to_cart(self, design_id, **kwargs):
        """Add customized product to cart"""
        design = request.env['packaging.design'].sudo().browse(design_id)
        
        # Get or create sale order (cart)
        order = request.website.sale_get_order(force_create=True)
        
        # Add line with design
        order._cart_update(
            product_id=design.product_id.id,
            line_id=None,
            add_qty=design.quantity,
            set_qty=0,
            packaging_design_id=design.id,
        )
        
        return {'success': True, 'cart_quantity': order.cart_quantity}
```

### 5. Frontend Design Tool (JavaScript)

```javascript
// design_tool.js
odoo.define('custom_packaging.design_tool', function (require) {
    'use strict';
    
    const publicWidget = require('web.public.widget');
    const ajax = require('web.ajax');
    
    publicWidget.registry.PackagingDesignTool = publicWidget.Widget.extend({
        selector: '.packaging-design-tool',
        events: {
            'click .upload-design-btn': '_onUploadClick',
            'change .design-file-input': '_onFileSelected',
            'click .save-design-btn': '_onSaveDesign',
            'click .add-to-cart-btn': '_onAddToCart',
            'change .material-select': '_onConfigChange',
            'change .color-count-input': '_onConfigChange',
            'change .print-type-select': '_onConfigChange',
            'change .quantity-input': '_onConfigChange',
        },
        
        start: function () {
            this._super.apply(this, arguments);
            this.canvas = null;
            this.canvasBack = null;  // For double-sided products
            this.designData = {};
            this.productData = this._loadProductData();
            this._initCanvas();
            
            // Initialize back canvas if product supports double-sided
            if (this.productData.supports_double_sided) {
                this._initBackCanvas();
            }
        },
        
        _initCanvas: function () {
            const canvasEl = this.$('.design-canvas')[0];
            this.canvas = new fabric.Canvas(canvasEl, {
                width: 800,
                height: 600,
                backgroundColor: '#ffffff',
            });
            
            // Load product mockup as background
            const mockupUrl = this.$el.data('mockup-url');
            fabric.Image.fromURL(mockupUrl, (img) => {
                this.canvas.setBackgroundImage(img, this.canvas.renderAll.bind(this.canvas), {
                    scaleX: this.canvas.width / img.width,
                    scaleY: this.canvas.height / img.height,
                });
            });
            
            // Enable object controls
            this.canvas.on('object:modified', this._onCanvasModified.bind(this));
        },
        
        _initBackCanvas: function () {
            const canvasBackEl = this.$('.design-canvas-back')[0];
            if (!canvasBackEl) return;
            
            this.canvasBack = new fabric.Canvas(canvasBackEl, {
                width: 800,
                height: 600,
                backgroundColor: '#ffffff',
            });
            
            // Load back mockup
            const mockupBackUrl = this.$el.data('mockup-back-url');
            if (mockupBackUrl) {
                fabric.Image.fromURL(mockupBackUrl, (img) => {
                    this.canvasBack.setBackgroundImage(img, this.canvasBack.renderAll.bind(this.canvasBack), {
                        scaleX: this.canvasBack.width / img.width,
                        scaleY: this.canvasBack.height / img.height,
                    });
                });
            }
            
            this.canvasBack.on('object:modified', this._onCanvasModified.bind(this));
        },
        
        _loadProductData: function () {
            return {
                id: parseInt(this.$el.data('product-id')),
                supports_pattern_repeat: this.$el.data('supports-pattern-repeat') === 'True',
                supports_double_sided: this.$el.data('supports-double-sided') === 'True',
                supports_die_cut: this.$el.data('supports-die-cut') === 'True',
                moq: parseInt(this.$el.data('moq')),
            };
        },
        
        _onFileSelected: function (ev) {
            const file = ev.target.files[0];
            if (!file) return;
            
            // Validate file
            if (!this._validateFile(file)) {
                this._showError('Invalid file. Please upload PNG, JPG, or PDF under 10MB.');
                return;
            }
            
            // Read file and upload
            const reader = new FileReader();
            reader.onload = (e) => {
                this._uploadFile(e.target.result, file.name);
            };
            reader.readAsDataURL(file);
        },
        
        _uploadFile: function (fileData, filename) {
            this.$('.upload-progress').show();
            
            ajax.jsonRpc('/shop/design/upload', 'call', {
                file_data: fileData.split(',')[1],  // Remove data:image/png;base64, prefix
                filename: filename,
            }).then((result) => {
                if (result.error) {
                    this._showError(result.error);
                    return;
                }
                
                this.designData.attachment_id = result.attachment_id;
                this._addImageToCanvas(result.url);
                this.$('.upload-progress').hide();
            });
        },
        
        _addImageToCanvas: function (imageUrl, targetCanvas) {
            const canvas = targetCanvas || this.canvas;
            
            fabric.Image.fromURL(imageUrl, (img) => {
                // Scale image to fit design area
                const scale = Math.min(
                    canvas.width * 0.6 / img.width,
                    canvas.height * 0.6 / img.height
                );
                
                img.set({
                    left: canvas.width / 2,
                    top: canvas.height / 2,
                    scaleX: scale,
                    scaleY: scale,
                    originX: 'center',
                    originY: 'center',
                });
                
                canvas.add(img);
                canvas.setActiveObject(img);
                canvas.renderAll();
                
                // Enable pattern repeat if supported
                if (this.productData.supports_pattern_repeat) {
                    this._showPatternRepeatOption(img, canvas);
                }
            });
        },
        
        _showPatternRepeatOption: function (img, canvas) {
            this.$('.pattern-repeat-controls').show();
            
            this.$('.enable-pattern-repeat').off('change').on('change', (e) => {
                if (e.target.checked) {
                    this._applyPatternRepeat(img, canvas);
                } else {
                    this._removePatternRepeat(canvas);
                }
            });
        },
        
        _applyPatternRepeat: function (img, canvas) {
            // Create pattern from image
            const patternSourceCanvas = new fabric.StaticCanvas();
            patternSourceCanvas.add(img.clone());
            patternSourceCanvas.renderAll();
            
            const pattern = new fabric.Pattern({
                source: patternSourceCanvas.getElement(),
                repeat: 'repeat',
            });
            
            // Create rectangle with pattern fill
            const rect = new fabric.Rect({
                left: 0,
                top: 0,
                width: canvas.width,
                height: canvas.height,
                fill: pattern,
            });
            
            canvas.add(rect);
            canvas.sendToBack(rect);
            canvas.renderAll();
            
            this.designData.uses_pattern_repeat = true;
        },
        
        _removePatternRepeat: function (canvas) {
            const objects = canvas.getObjects();
            objects.forEach(obj => {
                if (obj.type === 'rect' && obj.fill instanceof fabric.Pattern) {
                    canvas.remove(obj);
                }
            });
            canvas.renderAll();
            this.designData.uses_pattern_repeat = false;
        },
        
        _onConfigChange: function () {
            // Validate quantity against MOQ
            const quantity = parseInt(this.$('.quantity-input').val());
            if (quantity < this.productData.moq) {
                this.$('.moq-warning').show().text(
                    `Minimum order quantity is ${this.productData.moq} units`
                );
                this.$('.add-to-cart-btn').prop('disabled', true);
            } else {
                this.$('.moq-warning').hide();
                this.$('.add-to-cart-btn').prop('disabled', false);
            }
            
            this._updatePricing();
        },
        
        _updatePricing: function () {
            const config = this._getConfiguration();
            
            ajax.jsonRpc('/shop/design/calculate_price', 'call', config).then((result) => {
                this.$('.unit-price').text(result.currency + result.unit_price.toFixed(2));
                this.$('.total-price').text(result.currency + result.total_price.toFixed(2));
                
                // Display price breakdown
                if (result.breakdown) {
                    this._displayPriceBreakdown(result.breakdown);
                }
                
                // Display quantity tiers
                if (result.tiers && result.tiers.length > 0) {
                    this._displayQuantityTiers(result.tiers);
                }
            });
        },
        
        _displayPriceBreakdown: function (breakdown) {
            let html = '<div class="price-breakdown">';
            html += `<div>Base Price: ${breakdown.base_price.toFixed(2)}</div>`;
            if (breakdown.material_cost > 0) {
                html += `<div>Material: +${breakdown.material_cost.toFixed(2)}</div>`;
            }
            if (breakdown.color_cost > 0) {
                html += `<div>Additional Colors: +${breakdown.color_cost.toFixed(2)}</div>`;
            }
            if (breakdown.print_cost > 0) {
                html += `<div>Print Type: +${breakdown.print_cost.toFixed(2)}</div>`;
            }
            if (breakdown.double_sided_cost > 0) {
                html += `<div>Double-Sided: +${breakdown.double_sided_cost.toFixed(2)}</div>`;
            }
            html += '</div>';
            this.$('.price-breakdown-container').html(html);
        },
        
        _displayQuantityTiers: function (tiers) {
            let html = '<div class="quantity-tiers"><h4>Volume Discounts</h4>';
            tiers.forEach(tier => {
                html += `<div class="tier-item">
                    <span class="tier-qty">${tier.min_quantity}+ units:</span>
                    <span class="tier-price">$${tier.unit_price.toFixed(2)}/unit</span>
                    ${tier.discount_pct > 0 ? `<span class="tier-discount">(${tier.discount_pct.toFixed(0)}% off)</span>` : ''}
                </div>`;
            });
            html += '</div>';
            this.$('.quantity-tiers-container').html(html);
        },
        
        _getConfiguration: function () {
            return {
                product_id: this.productData.id,
                material_id: parseInt(this.$('.material-select').val()),
                color_count: parseInt(this.$('.color-count-input').val()),
                print_type_id: parseInt(this.$('.print-type-select').val()),
                quantity: parseInt(this.$('.quantity-input').val()),
                is_double_sided: this.$('.double-sided-checkbox').is(':checked'),
            };
        },
        
        _onSaveDesign: function () {
            const config = this._getConfiguration();
            config.canvas_json = JSON.stringify(this.canvas.toJSON());
            config.attachment_id = this.designData.attachment_id;
            config.uses_pattern_repeat = this.designData.uses_pattern_repeat || false;
            
            // Save back canvas if double-sided
            if (this.canvasBack && config.is_double_sided) {
                config.canvas_json_back = JSON.stringify(this.canvasBack.toJSON());
                config.attachment_back_id = this.designData.attachment_back_id;
            }
            
            // Save die-cut shape if applicable
            if (this.productData.supports_die_cut) {
                config.die_cut_shape = this.$('.die-cut-select').val();
            }
            
            ajax.jsonRpc('/shop/design/save', 'call', {
                product_id: config.product_id,
                design_data: config,
            }).then((result) => {
                this.designData.design_id = result.design_id;
                this._showSuccess('Design saved successfully!');
            });
        },
        
        _onAddToCart: function () {
            if (!this.designData.design_id) {
                this._showError('Please save your design first.');
                return;
            }
            
            ajax.jsonRpc('/shop/design/' + this.designData.design_id + '/add_to_cart', 'call', {})
                .then((result) => {
                    if (result.success) {
                        window.location.href = '/shop/cart';
                    }
                });
        },
        
        _validateFile: function (file) {
            const validTypes = ['image/png', 'image/jpeg', 'application/pdf'];
            const maxSize = 10 * 1024 * 1024; // 10MB
            
            return validTypes.includes(file.type) && file.size <= maxSize;
        },
    });
    
    return publicWidget.registry.PackagingDesignTool;
});
```

## Data Models

### Entity Relationship Diagram

```
product.template (extended)
    ├── is_custom_packaging: Boolean
    ├── packaging_category: Selection
    ├── minimum_order_qty: Integer
    ├── material_ids: Many2many → product.packaging.material
    ├── certification_ids: Many2many → packaging.certification
    └── design_template_ids: One2many → product.packaging.template

product.packaging.material
    ├── name: Char
    ├── price_multiplier: Float
    └── certification_ids: Many2many → packaging.certification

packaging.certification
    ├── name: Char
    ├── badge_image: Binary
    └── verification_url: Char

packaging.design
    ├── product_id: Many2one → product.product
    ├── sale_order_line_id: Many2one → sale.order.line
    ├── design_file_id: Many2one → ir.attachment
    ├── canvas_json: Text
    ├── material_id: Many2one → product.packaging.material
    ├── proof_status: Selection
    └── proof_file_id: Many2one → ir.attachment

sale.order.line (extended)
    └── packaging_design_id: Many2one → packaging.design
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property Reflection

After analyzing all acceptance criteria, I've identified several areas where properties can be consolidated:

**Redundancy Analysis:**

1. **Price Calculation Properties (7.1-7.4)**: These four properties all test the same underlying pricing engine with different inputs. They can be combined into a single comprehensive property that tests price recalculation for any configuration change.

2. **MOQ Validation Properties (2.2-2.4)**: Properties 2.2 and 2.3 both test rejection of below-MOQ quantities, while 2.4 tests acceptance of valid quantities. These can be combined into a single property about MOQ enforcement.

3. **Design Tool Manipulation Properties (5.3-5.5)**: Drag-and-drop, resize, and rotation are all canvas manipulation operations that modify object properties. These can be combined into a single property about canvas object manipulation.

4. **UI Rendering Properties**: Many properties test that specific elements appear in rendered HTML (3.2, 3.5, 4.1, 6.4, 7.5, 8.4). These can be consolidated into fewer properties that test rendering of configuration-specific elements.

5. **File Upload Properties (4.3, 4.7)**: Both test file validation - one for size/type, another for format/size/resolution. These overlap and can be combined.

6. **Cart Persistence Properties (8.2, 8.7)**: Both test that design data is stored with cart items. These can be combined into a single property about cart data persistence.

**Consolidated Property List:**

After reflection, we'll focus on these unique, high-value properties:
- Product configuration storage and retrieval
- MOQ validation (combined 2.2-2.4)
- Material pricing calculation
- Dynamic pricing engine (combined 7.1-7.4)
- File upload validation (combined 4.3, 4.7)
- Canvas serialization round-trip (5.6)
- Canvas object manipulation (combined 5.3-5.5)
- Cart design data persistence (combined 8.2, 8.7)
- Proof generation with complete data
- Order workflow state transitions
- Access control enforcement
- Translation completeness

### Correctness Properties

Based on the prework analysis and property reflection, here are the key correctness properties for the custom packaging system:

#### Property 1: Product Configuration Persistence

*For any* packaging product with configured materials, certifications, and MOQ, creating and retrieving the product should preserve all configuration data exactly.

**Validates: Requirements 1.2, 1.3, 2.1, 3.1**

#### Property 1.5: Product-Specific Feature Configuration

*For any* packaging product of a specific category (tissue, mailer, box, sticker, tape, tote), the system should automatically enable category-appropriate features (pattern repeat for tissue, double-sided for mailers/boxes, die-cut for stickers/tape, organic cotton for totes) and set category-specific MOQ defaults.

**Validates: Requirements 1.5.1, 1.5.2, 1.5.3, 1.5.4, 1.5.5**

#### Property 2: MOQ Enforcement with Product-Specific Values

*For any* packaging product with a minimum order quantity N (where N varies by category: Tissue=250, Mailers=100-250, Stickers/Tape=250, Boxes=100, Totes=50), attempting to add quantities less than N to the cart should be rejected, and quantities greater than or equal to N should be accepted.

**Validates: Requirements 2.2, 2.3, 2.4, 2.5, 2.6**

#### Property 3: Material Price Calculation

*For any* product with base price P and material with multiplier M and additional cost A, selecting that material should result in a price of (P × M) + A (before other adjustments).

**Validates: Requirements 3.4**

#### Property 4: Dynamic Pricing Calculation with Tiers

*For any* product configuration (quantity Q, material M, color count C, print type T, double-sided D), the final unit price should equal: tier_base_price(Q) + material_cost(M) + (C - 1) × color_price + print_type_cost(T) + (D ? double_sided_cost : 0), where tier_base_price is determined by the highest quantity tier where Q >= tier.min_quantity.

**Validates: Requirements 7.1, 7.2, 7.3, 7.4, 7.7, 7.8, 7.9**

#### Property 5: File Upload Validation

*For any* uploaded file, the system should accept files of type PNG, JPG, PDF, or SVG under 10MB with minimum 300 DPI resolution, and reject all other files with appropriate error messages.

**Validates: Requirements 4.3, 4.7**

#### Property 6: Canvas Serialization Round-Trip with Pattern Repeat

*For any* design canvas state with uploaded images, positioning, scaling, rotation, and optional pattern repeat applied, serializing to JSON and deserializing should produce an equivalent canvas state with all properties preserved.

**Validates: Requirements 5.6**

#### Property 7: Canvas Object Manipulation

*For any* design object on the canvas, applying drag (position change), resize (scale change), or rotation operations should update the corresponding object properties (left/top, scaleX/scaleY, angle) accordingly, and pattern repeat should tile the design across the canvas when enabled.

**Validates: Requirements 5.3, 5.4, 5.5, 4.5, 4.9**

#### Property 8: Cart Design Data Persistence with Double-Sided Support

*For any* customized product with design file(s), canvas JSON (front and optionally back), material, colors, print type, double-sided flag, and pattern repeat flag, adding to cart should store all design data as a packaging.design record linked to the cart line item, and this data should be retrievable with all properties intact.

**Validates: Requirements 8.2, 8.7**

#### Property 9: Cart Session Persistence

*For any* logged-in customer with items in cart, logging out and logging back in should preserve all cart items with their customization data.

**Validates: Requirements 8.6**

#### Property 10: Unique Cart Line Items

*For any* two product configurations with different customization options (material, colors, or design), adding both to cart should create two separate line items, not merge them.

**Validates: Requirements 8.3**

#### Property 11: Proof Generation Completeness

*For any* customized product in an order, generating a design proof should produce a PDF containing the product mockup with applied design (including back side if double-sided), material name, color count, print type, quantity, sustainability certifications, and pricing breakdown.

**Validates: Requirements 9.1, 9.2, 9.3**

#### Property 12: Proof Approval State Transition

*For any* order with proof status 'pending', approving the proof should change the status to 'approved' and mark the order as ready for production, while rejecting should change status to 'rejected' and allow design modifications.

**Validates: Requirements 9.6, 9.7**

#### Property 13: Checkout Validation

*For any* cart with customized products, proceeding to checkout should validate that all required customization fields (design file, material, quantity >= MOQ) are complete, rejecting incomplete configurations.

**Validates: Requirements 10.2**

#### Property 14: Order Creation with Attachments

*For any* completed checkout with N customized products, the system should create a sale order with N line items, each linked to its packaging.design record and associated design file attachments.

**Validates: Requirements 10.3, 10.4**

#### Property 15: Template Loading

*For any* product with associated design templates, selecting a template in the design tool should load the template's canvas JSON into the canvas, allowing further modifications.

**Validates: Requirements 11.4, 11.5**

#### Property 16: Access Control Enforcement

*For any* user without administrator permissions, attempting to create, update, or delete product records should be denied with an access error.

**Validates: Requirements 14.2, 14.3**

#### Property 17: Customer Data Isolation

*For any* customer user, querying orders or designs should return only records owned by that customer, never records belonging to other customers.

**Validates: Requirements 14.4**

#### Property 18: Filename Sanitization

*For any* uploaded file with a filename containing special characters (/, \, <, >, :, ", |, ?, *, script tags), the system should sanitize the filename by removing or replacing dangerous characters before storage.

**Validates: Requirements 14.6**

#### Property 19: Translation Completeness

*For any* supported language L, switching to language L should display all product names, descriptions, UI labels, and error messages in language L (no untranslated text).

**Validates: Requirements 12.2**

#### Property 20: SEO URL Format

*For any* product with name N, the generated product URL should be in the format /shop/product/{slug} where slug is a lowercase, hyphenated version of N with special characters removed.

**Validates: Requirements 12.3**

#### Property 21: Responsive Layout Adaptation

*For any* page in the system, rendering at mobile viewport width (< 768px) should apply mobile-specific CSS classes and adjust layouts to single-column format.

**Validates: Requirements 13.2**

#### Property 22: Thumbnail Generation

*For any* uploaded design image, the system should generate a thumbnail version with maximum dimensions of 200x200 pixels while maintaining aspect ratio.

**Validates: Requirements 15.2**

## Error Handling

### File Upload Errors

```python
class DesignFileUploadError(UserError):
    """Raised when design file upload fails validation"""
    pass

# In upload handler:
def _validate_design_file(self, file_data, filename):
    # Check file size
    if len(file_data) > 10 * 1024 * 1024:
        raise DesignFileUploadError(_('File size exceeds 10MB limit'))
    
    # Check file type
    allowed_types = ['image/png', 'image/jpeg', 'application/pdf']
    mime_type = magic.from_buffer(file_data, mime=True)
    if mime_type not in allowed_types:
        raise DesignFileUploadError(_('Invalid file type. Only PNG, JPG, and PDF are allowed'))
    
    # Check image resolution for raster images
    if mime_type.startswith('image/'):
        image = Image.open(io.BytesIO(file_data))
        if image.width < 300 or image.height < 300:
            raise DesignFileUploadError(_('Image resolution too low. Minimum 300x300 pixels required'))
```

### MOQ Validation Errors

```python
class MOQValidationError(ValidationError):
    """Raised when quantity is below minimum order quantity"""
    pass

# In cart update:
def _check_moq(self, product, quantity):
    if product.is_custom_packaging and quantity < product.minimum_order_qty:
        raise MOQValidationError(
            _('Minimum order quantity for %s is %d units') % (product.name, product.minimum_order_qty)
        )
```

### Pricing Calculation Errors

```python
# Graceful handling of missing configuration
def _calculate_price(self, product, material, color_count, print_type, quantity):
    try:
        base_price = product.list_price
        
        # Apply material multiplier with default
        material_multiplier = material.price_multiplier if material else 1.0
        price = base_price * material_multiplier
        
        # Apply color pricing with validation
        if color_count > product.max_colors:
            _logger.warning('Color count %d exceeds maximum %d for product %s', 
                          color_count, product.max_colors, product.name)
            color_count = product.max_colors
        
        price += (color_count - 1) * product.price_per_additional_color
        
        # Apply print type cost
        if print_type:
            price += print_type.additional_cost
        
        # Apply quantity discounts
        for tier in product.quantity_price_tier_ids.sorted(key=lambda t: t.min_quantity, reverse=True):
            if quantity >= tier.min_quantity:
                price *= tier.discount_multiplier
                break
        
        return price
    
    except Exception as e:
        _logger.error('Error calculating price for product %s: %s', product.name, str(e))
        return product.list_price  # Fallback to base price
```

### Canvas State Errors

```python
# Handle corrupted canvas JSON
def _load_canvas_state(self, canvas_json):
    try:
        canvas_data = json.loads(canvas_json)
        return canvas_data
    except json.JSONDecodeError:
        _logger.error('Invalid canvas JSON data')
        return {'objects': [], 'background': None}  # Return empty canvas
```

### Proof Generation Errors

```python
# Handle proof generation failures
def _generate_design_proof(self, design):
    try:
        # Generate PDF proof
        pdf_content = self._render_proof_pdf(design)
        
        # Create attachment
        attachment = self.env['ir.attachment'].create({
            'name': f'proof_{design.id}.pdf',
            'datas': base64.b64encode(pdf_content),
            'res_model': 'packaging.design',
            'res_id': design.id,
        })
        
        design.proof_file_id = attachment
        design.proof_status = 'pending'
        
    except Exception as e:
        _logger.error('Failed to generate proof for design %s: %s', design.id, str(e))
        design.proof_status = 'draft'
        raise UserError(_('Failed to generate design proof. Please try again or contact support.'))
```

## Testing Strategy

### Dual Testing Approach

The custom packaging module requires both unit tests and property-based tests to ensure comprehensive coverage:

**Unit Tests** focus on:
- Specific examples of product configurations
- Edge cases (empty cart, missing materials, zero quantity)
- Integration points between modules (product → website_sale → sale)
- Error conditions (invalid files, permission denied, missing data)
- UI rendering for specific scenarios

**Property-Based Tests** focus on:
- Universal properties across all inputs (pricing calculations, MOQ validation)
- Comprehensive input coverage through randomization (all material combinations, quantity ranges)
- Round-trip properties (canvas serialization, cart persistence)
- Invariants (price always positive, quantity always >= MOQ in cart)

### Property-Based Testing Configuration

**Framework**: Hypothesis (Python property-based testing library)

**Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with feature name and property number
- Tag format: `# Feature: custom-packaging-ecommerce, Property N: {property_text}`

**Example Property Test Structure**:

```python
from hypothesis import given, strategies as st
import hypothesis

# Configure Hypothesis
hypothesis.settings.register_profile("ci", max_examples=100)
hypothesis.settings.load_profile("ci")

class TestPackagingProperties(TransactionCase):
    
    @given(
        base_price=st.floats(min_value=1.0, max_value=1000.0),
        material_multiplier=st.floats(min_value=0.5, max_value=3.0),
        color_count=st.integers(min_value=1, max_value=6),
        quantity=st.integers(min_value=1, max_value=10000),
    )
    def test_property_4_dynamic_pricing(self, base_price, material_multiplier, color_count, quantity):
        """
        Feature: custom-packaging-ecommerce, Property 4: Dynamic Pricing Calculation
        
        For any product configuration, changing any parameter should trigger
        price recalculation with correct formula application.
        """
        # Create test product
        product = self.env['product.template'].create({
            'name': 'Test Product',
            'is_custom_packaging': True,
            'list_price': base_price,
            'price_per_additional_color': 5.0,
        })
        
        material = self.env['product.packaging.material'].create({
            'name': 'Test Material',
            'price_multiplier': material_multiplier,
        })
        
        # Calculate price
        calculated_price = product._calculate_custom_price(
            material=material,
            color_count=color_count,
            quantity=quantity,
        )
        
        # Expected price formula
        expected_price = base_price * material_multiplier + (color_count - 1) * 5.0
        
        # Assert price matches formula (within floating point tolerance)
        self.assertAlmostEqual(calculated_price, expected_price, places=2)
        
        # Assert price is always positive
        self.assertGreater(calculated_price, 0)
    
    @given(
        moq=st.integers(min_value=1, max_value=1000),
        quantity=st.integers(min_value=1, max_value=2000),
    )
    def test_property_2_moq_enforcement(self, moq, quantity):
        """
        Feature: custom-packaging-ecommerce, Property 2: MOQ Enforcement
        
        For any product with MOQ N, quantities < N should be rejected,
        quantities >= N should be accepted.
        """
        product = self.env['product.product'].create({
            'name': 'Test Product',
            'is_custom_packaging': True,
            'minimum_order_qty': moq,
        })
        
        order = self.env['sale.order'].create({
            'partner_id': self.env.ref('base.partner_demo').id,
        })
        
        if quantity < moq:
            # Should raise validation error
            with self.assertRaises(ValidationError):
                order._cart_update(
                    product_id=product.id,
                    add_qty=quantity,
                )
        else:
            # Should succeed
            line = order._cart_update(
                product_id=product.id,
                add_qty=quantity,
            )
            self.assertEqual(line.product_uom_qty, quantity)
```

### Unit Test Examples

```python
class TestPackagingUnit(TransactionCase):
    
    def test_product_configuration_storage(self):
        """Test that packaging product configuration is stored correctly"""
        material = self.env['product.packaging.material'].create({
            'name': 'Recycled Paper',
            'price_multiplier': 1.2,
        })
        
        cert = self.env['packaging.certification'].create({
            'name': 'FSC Certified',
            'code': 'FSC',
        })
        
        product = self.env['product.template'].create({
            'name': 'Custom Tissue Paper',
            'is_custom_packaging': True,
            'packaging_category': 'tissue',
            'minimum_order_qty': 500,
            'material_ids': [(6, 0, [material.id])],
            'certification_ids': [(6, 0, [cert.id])],
        })
        
        # Verify all fields stored correctly
        self.assertTrue(product.is_custom_packaging)
        self.assertEqual(product.packaging_category, 'tissue')
        self.assertEqual(product.minimum_order_qty, 500)
        self.assertIn(material, product.material_ids)
        self.assertIn(cert, product.certification_ids)
    
    def test_empty_cart_checkout_validation(self):
        """Test that empty cart cannot proceed to checkout"""
        order = self.env['sale.order'].create({
            'partner_id': self.env.ref('base.partner_demo').id,
        })
        
        with self.assertRaises(ValidationError):
            order.action_confirm()
    
    def test_design_file_upload_invalid_type(self):
        """Test that invalid file types are rejected"""
        controller = DesignToolController()
        
        # Try to upload .exe file
        result = controller.upload_design_file(
            file_data=b'MZ\x90\x00',  # EXE header
            filename='malware.exe',
        )
        
        self.assertIn('error', result)
        self.assertIn('Invalid file', result['error'])
```

### Integration Test Examples

```python
class TestPackagingIntegration(HttpCase):
    
    def test_full_customization_workflow(self):
        """Test complete workflow from product page to order"""
        # 1. Navigate to product page
        product = self.env.ref('custom_packaging.demo_tissue_paper')
        self.url_open(f'/shop/product/{product.id}')
        
        # 2. Click customize button
        self.browser_js(
            url_path=f'/shop/product/{product.id}',
            code="document.querySelector('.customize-btn').click();",
        )
        
        # 3. Upload design
        # 4. Configure options
        # 5. Add to cart
        # 6. Proceed to checkout
        # 7. Verify order created with design
        
        # This would be a full browser automation test
```

### Test Coverage Goals

- **Unit Tests**: 80% code coverage minimum
- **Property Tests**: All 22 correctness properties implemented
- **Integration Tests**: All major user workflows covered
- **Performance Tests**: Page load < 2s, design tool responsive < 100ms

### Continuous Integration

```yaml
# .gitlab-ci.yml or similar
test:
  script:
    - odoo-bin -d test_db -i custom_packaging --test-enable --stop-after-init
    - pytest tests/ --hypothesis-profile=ci --cov=custom_packaging --cov-report=html
```

## Deployment Considerations

### Database Migrations

When installing the module, the following database changes occur:

1. New tables created:
   - `product_packaging_material`
   - `packaging_certification`
   - `packaging_design`
   - `product_packaging_template`
   - `product_print_type`

2. Existing tables extended:
   - `product_template` (new columns added)
   - `product_product` (new columns added)
   - `sale_order_line` (new columns added)

3. Many2many relationship tables created automatically by Odoo

### File Storage Requirements

- Design files stored in Odoo's filestore (typically `~/.local/share/Odoo/filestore/{db_name}`)
- Estimated storage: 2-5 MB per design (depends on image resolution)
- Proof PDFs: 1-2 MB per proof
- Recommend monitoring filestore size and implementing cleanup for old/cancelled orders

### Performance Optimization

1. **Database Indexes**: Add indexes on frequently queried fields
   ```sql
   CREATE INDEX idx_packaging_design_product ON packaging_design(product_id);
   CREATE INDEX idx_packaging_design_order_line ON packaging_design(sale_order_line_id);
   CREATE INDEX idx_product_template_packaging ON product_template(is_custom_packaging);
   ```

2. **Caching**: Enable Odoo's built-in caching for product images and static assets

3. **CDN**: Serve static assets (Fabric.js, product mockups) from CDN in production

4. **Async Processing**: Use Odoo's queue_job module for:
   - Proof PDF generation
   - Large file uploads
   - Email sending

### Security Hardening

1. **File Upload Validation**: Implement virus scanning for uploaded files
2. **Rate Limiting**: Limit design uploads to prevent abuse
3. **CSRF Protection**: Ensure all forms have CSRF tokens
4. **SQL Injection**: Use Odoo ORM exclusively (no raw SQL)
5. **XSS Prevention**: Sanitize all user inputs, especially filenames

### Monitoring and Logging

```python
import logging
_logger = logging.getLogger(__name__)

# Log important events
_logger.info('Design uploaded: product=%s, user=%s, size=%d', 
             product.name, request.env.user.name, file_size)

_logger.warning('MOQ validation failed: product=%s, quantity=%d, moq=%d',
                product.name, quantity, product.minimum_order_qty)

_logger.error('Proof generation failed: design=%s, error=%s',
              design.id, str(e))
```

### Backup Strategy

- Regular database backups (daily minimum)
- Filestore backups (design files and proofs)
- Test restore procedures regularly
- Keep backups for at least 90 days (or per business requirements)

## Future Enhancements

### Phase 2 Features (Not in Current Scope)

1. **Advanced Design Tools**:
   - Text overlay with custom fonts
   - Color picker for background colors
   - Filters and effects (grayscale, sepia, etc.)
   - Multi-page designs for boxes

2. **3D Product Previews**:
   - Three.js integration for 3D mockups
   - Rotate and zoom 3D models
   - More realistic material rendering

3. **Design Library**:
   - Save designs for reuse
   - Share designs with team members
   - Design version history

4. **Bulk Ordering**:
   - Upload CSV for multiple products
   - Quantity discounts for multiple items
   - Corporate account management

5. **Production Integration**:
   - Direct integration with printing equipment
   - Real-time production status tracking
   - Quality control checkpoints

6. **Analytics Dashboard**:
   - Popular products and materials
   - Average order value
   - Design complexity metrics
   - Customer behavior analysis

### Scalability Considerations

- Implement Redis caching for high-traffic scenarios
- Consider microservices architecture for design tool if traffic grows significantly
- Implement CDN for global distribution
- Database read replicas for reporting queries

## Conclusion

This design provides a comprehensive architecture for the custom packaging e-commerce module, extending Odoo 19's core functionality while maintaining compatibility and following best practices. The modular structure allows for incremental development and testing, with clear separation of concerns between backend logic, frontend interactions, and data storage.

The property-based testing approach ensures correctness across a wide range of inputs, while unit tests cover specific scenarios and edge cases. The design emphasizes user experience, performance, and security, making it suitable for production deployment.

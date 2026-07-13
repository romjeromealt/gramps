# Fan Chart Consolidation - Implementation Plan

## Overview

This document provides a step-by-step implementation plan for consolidating the Fan Chart implementations in Gramps, addressing request 0012927 to extend Report choices while reducing code duplication.

## Current Architecture Issues

### 1. Code Duplication Metrics
- **Widget files**: 3 separate files with ~4,123 lines total
- **View plugins**: 3 files with duplicated initialization logic
- **Gramplets**: 3 files with duplicated setup code
- **Configuration**: Each type maintains its own config keys

### 2. Inconsistent Features
- Different background options available
- Different form options
- Different name display options
- Report only supports ancestral charts

## Phase 1: Configuration Consolidation (Week 1-2)

### Step 1.1: Create Unified Configuration Module

**New file**: `gramps/gui/widgets/fanchartconfig.py`

```python
# -------------------------------------------------------------------------
#
# Fan Chart Configuration Module
#
# -------------------------------------------------------------------------

"""
Unified configuration management for all fan chart types.
This module provides consistent configuration options across all fan chart
implementations (widgets, views, gramplets, reports).
"""

from gramps.gen.const import (
    PIXELS_PER_GENERATION,
    BORDER_EDGE_WIDTH,
    CHILDRING_WIDTH,
    TRANSLATE_PX,
    PAD_PX,
    PAD_TEXT,
    BACKGROUND_SCHEME1,
    BACKGROUND_SCHEME2,
    BACKGROUND_GENDER,
    BACKGROUND_WHITE,
    BACKGROUND_GRAD_GEN,
    BACKGROUND_GRAD_AGE,
    BACKGROUND_SINGLE_COLOR,
    BACKGROUND_GRAD_PERIOD,
    GENCOLOR,
    MAX_AGE,
    GRADIENTSCALE,
    FORM_CIRCLE,
    FORM_HALFCIRCLE,
    FORM_QUADRANT,
    COLLAPSED,
    NORMAL,
    EXPANDED,
)

# Angle calculation algorithms
ANGLE_CHEQUI = 0  # Algorithm with homogeneous children distribution
ANGLE_WEIGHT = 1  # Algorithm for angle computation based on nr of descendants

# Box types
TYPE_BOX_NORMAL = 0
TYPE_BOX_FAMILY = 1

# Chart types
CHART_ANCESTRAL = "ancestral"
CHART_DESCENDANT = "descendant"
CHART_TWOWAY = "twoway"

# Chart type constants for backward compatibility
TYPE_ASCENDANCE = 0
TYPE_DESCENDANCE = 1

class FanChartConfig:
    """
    Unified configuration container for fan charts.
    """
    
    def __init__(self, chart_type=CHART_ANCESTRAL):
        """
        Initialize fan chart configuration.
        
        :param chart_type: Type of fan chart ('ancestral', 'descendant', 'twoway')
        """
        self.chart_type = chart_type
        self._reset_to_defaults()
    
    def _reset_to_defaults(self):
        """Reset all configuration to default values."""
        # Common settings for all chart types
        self.maxgen = 9
        self.background = BACKGROUND_GRAD_GEN
        self.radialtext = True
        self.twolinename = True
        self.flipupsidedownname = True
        self.font = "Sans"
        self.form = FORM_CIRCLE
        self.showid = False
        self.grad_start = "#0000FF"
        self.grad_end = "#FF0000"
        self.alpha_filter = 0.2
        self.filter = None
        
        # Ancestral-specific settings
        self.childring = True
        
        # Descendant-specific settings
        self.angle_algo = ANGLE_WEIGHT
        self.dupcolor = "#888a85"
        
        # Two-way specific settings
        self.generations_asc = 4
        self.generations_desc = 4
        self.background_gradient = True
        
        # Widget-specific settings
        self.pixels_per_generation = PIXELS_PER_GENERATION
        self.border_edge_width = BORDER_EDGE_WIDTH
        self.childring_width = CHILDRING_WIDTH
        self.translate_px = TRANSLATE_PX
        self.pad_px = PAD_PX
        self.pad_text = PAD_TEXT
    
    def apply_config_dict(self, config_dict):
        """
        Apply configuration from a dictionary.
        
        :param config_dict: Dictionary of configuration values
        """
        for key, value in config_dict.items():
            if hasattr(self, key):
                setattr(self, key, value)
    
    def to_dict(self):
        """Convert configuration to dictionary."""
        return {
            'maxgen': self.maxgen,
            'background': self.background,
            'radialtext': self.radialtext,
            'twolinename': self.twolinename,
            'flipupsidedownname': self.flipupsidedownname,
            'font': self.font,
            'form': self.form,
            'showid': self.showid,
            'grad_start': self.grad_start,
            'grad_end': self.grad_end,
            'alpha_filter': self.alpha_filter,
            'childring': self.childring,
            'angle_algo': self.angle_algo,
            'dupcolor': self.dupcolor,
            'generations_asc': self.generations_asc,
            'generations_desc': self.generations_desc,
            'background_gradient': self.background_gradient,
        }
    
    def get_config_for_type(self, chart_type):
        """
        Get appropriate configuration values for a specific chart type.
        
        :param chart_type: Type of fan chart
        :returns: Dictionary of relevant configuration values
        """
        config = self.to_dict()
        
        if chart_type == CHART_ANCESTRAL:
            # For ancestral, use maxgen
            config['generations'] = config['maxgen']
        elif chart_type == CHART_DESCENDANT:
            # For descendant, use maxgen
            config['generations'] = config['maxgen']
        elif chart_type == CHART_TWOWAY:
            # For two-way, use both generations_asc and generations_desc
            pass
            
        return config

# Configuration presets for different chart types
CONFIG_PRESETS = {
    CHART_ANCESTRAL: {
        'maxgen': 9,
        'childring': True,
    },
    CHART_DESCENDANT: {
        'maxgen': 6,
        'childring': False,
        'angle_algo': ANGLE_WEIGHT,
        'dupcolor': "#888a85",
    },
    CHART_TWOWAY: {
        'generations_asc': 4,
        'generations_desc': 4,
        'childring': False,
        'background_gradient': True,
        'angle_algo': ANGLE_WEIGHT,
        'dupcolor': "#888a85",
    }
}

# Legacy configuration keys mapping
LEGACY_CONFIG_MAP = {
    # Ancestral view
    "interface.fanview-maxgen": "maxgen",
    "interface.fanview-background": "background", 
    "interface.fanview-childrenring": "childring",
    "interface.fanview-radialtext": "radialtext",
    "interface.fanview-twolinename": "twolinename",
    "interface.fanview-flipupsidedownname": "flipupsidedownname",
    "interface.fanview-font": "font",
    "interface.fanview-form": "form",
    "interface.fanview-showid": "showid",
    "interface.color-start-grad": "grad_start",
    "interface.color-end-grad": "grad_end",
    
    # Descendant view
    "interface.angle-algorithm": "angle_algo",
    "interface.duplicate-color": "dupcolor",
    
    # Two-way view
    "interface.fanview-maxgen-asc": "generations_asc",
    "interface.fanview-maxgen-desc": "generations_desc",
    "interface.fanview-background-gradient": "background_gradient",
}
```

### Step 1.2: Update Constants in const.py

Add chart type constants to the existing constants file for consistency.

### Step 1.3: Create Migration Script

Create a script to migrate existing user configurations to the new unified system.

## Phase 2: Base Widget Enhancement (Week 3-4)

### Step 2.1: Refactor FanChartBaseWidget

Enhance the base widget to support all chart types:

```python
class FanChartBaseWidget(Gtk.DrawingArea):
    """Unified base widget for all fan chart types"""

    CENTER = 60  # pixel radius of center, changes per fanchart

    def __init__(self, dbstate, uistate, callback_popup=None, chart_type=CHART_ANCESTRAL):
        """
        Initialize the fan chart widget.
        
        :param dbstate: Database state
        :param uistate: UI state
        :param callback_popup: Callback for popup menu
        :param chart_type: Type of fan chart ('ancestral', 'descendant', 'twoway')
        """
        Gtk.DrawingArea.__init__(self)
        
        # Store chart type
        self.chart_type = chart_type
        self.dbstate = dbstate
        self.uistate = uistate
        self.on_popup = callback_popup
        
        # Configuration
        self.config = FanChartConfig(chart_type)
        
        # Common data structures
        self.radialtext = True
        self.textcolor = (0, 0, 0)  # Default text color
        self.angle = {}
        self.childrenroot = []
        self.rootangle_rad = []
        self.generations = 8
        self.filter = None
        self.alpha_filter = 0.5
        self.translating = False
        self.showid = False
        self.flipupsidedownname = True
        self.dupcolor = None
        self.twolinename = False
        self.surface = None
        self.goto = None
        self.last_x, self.last_y = None, None
        self.fontdescr = "Sans"
        self.fontsize = 8
        
        # Type-specific data structures
        if chart_type in [CHART_DESCENDANT, CHART_TWOWAY]:
            self.gen2people = {}
            self.gen2fam = {}
            self.handle2desc = {}
            self.famhandle2desc = {}
            self.handle2fam = {}
            self.innerring = []
        
        # Initialize based on chart type
        self._initialize_for_type()
        
        # Common setup
        self._setup_common()
        self.reset()
        self.set_size_request(120, 120)
        self.maxperiod = 0
    
    def _initialize_for_type(self):
        """Initialize type-specific properties and data structures."""
        if self.chart_type == CHART_ANCESTRAL:
            self.CENTER = 60
            self.childring = True
        elif self.chart_type == CHART_DESCENDANT:
            self.CENTER = 60  # Larger center for partner
            self.childring = False
        elif self.chart_type == CHART_TWOWAY:
            self.CENTER = 50  # Larger center for both directions
            self.childring = False
    
    def set_config(self, config):
        """
        Set configuration for the widget.
        
        :param config: FanChartConfig object or dictionary
        """
        if isinstance(config, dict):
            self.config.apply_config_dict(config)
        elif isinstance(config, FanChartConfig):
            self.config = config
        else:
            raise ValueError("Config must be dict or FanChartConfig")
        
        # Apply configuration to widget properties
        self._apply_config_to_properties()
    
    def _apply_config_to_properties(self):
        """Apply configuration values to widget properties."""
        # Common properties
        self.generations = self.config.maxgen
        self.background = self.config.background
        self.radialtext = self.config.radialtext
        self.twolinename = self.config.twolinename
        self.flipupsidedownname = self.config.flipupsidedownname
        self.fontdescr = self.config.font
        self.form = self.config.form
        self.showid = self.config.showid
        self.grad_start = self.config.grad_start
        self.grad_end = self.config.grad_end
        self.alpha_filter = self.config.alpha_filter
        self.filter = self.config.filter
        
        # Type-specific properties
        if self.chart_type == CHART_ANCESTRAL:
            self.childring = self.config.childring
        elif self.chart_type in [CHART_DESCENDANT, CHART_TWOWAY]:
            self.anglealgo = self.config.angle_algo
            self.dupcolor = hex_to_rgb(self.config.dupcolor)
        
        if self.chart_type == CHART_TWOWAY:
            self.generations_asc = self.config.generations_asc
            self.generations_desc = self.config.generations_desc
            self.background_gradient = self.config.background_gradient
```

### Step 2.2: Create Type-Specific Mixins

Instead of multiple inheritance, use mixins for type-specific functionality:

```python
class AncestralMixin:
    """Mixin for ancestral fan chart functionality."""
    
    def set_values(self, root_person_handle, maxgen, background, childring, 
                   flipupsidedownname, twolinename, radialtext, fontdescr,
                   grad_start, grad_end, filtr, alpha_filter, form, showid):
        """Set values for ancestral fan chart."""
        self.rootpersonh = root_person_handle
        self.config.maxgen = maxgen
        self.config.background = background
        self.config.childring = childring
        self.config.flipupsidedownname = flipupsidedownname
        self.config.twolinename = twolinename
        self.config.radialtext = radialtext
        self.config.font = fontdescr
        self.config.grad_start = grad_start
        self.config.grad_end = grad_end
        self.config.filter = filtr
        self.config.alpha_filter = alpha_filter
        self.config.form = form
        self.config.showid = showid
        self._apply_config_to_properties()
    
    def set_generations(self):
        """Set up generations for ancestral chart."""
        # Ancestral-specific implementation
        pass

class DescendantMixin:
    """Mixin for descendant fan chart functionality."""
    
    def set_values(self, root_person_handle, maxgen, flipupsidedownname, 
                   twolinename, background, fontdescr, grad_start, grad_end,
                   filtr, alpha_filter, form, angle_algo, dupcolor, showid):
        """Set values for descendant fan chart."""
        self.rootpersonh = root_person_handle
        self.config.maxgen = maxgen
        self.config.flipupsidedownname = flipupsidedownname
        self.config.twolinename = twolinename
        self.config.background = background
        self.config.font = fontdescr
        self.config.grad_start = grad_start
        self.config.grad_end = grad_end
        self.config.filter = filtr
        self.config.alpha_filter = alpha_filter
        self.config.form = form
        self.config.angle_algo = angle_algo
        self.config.dupcolor = dupcolor
        self.config.showid = showid
        self._apply_config_to_properties()

class TwoWayMixin(AncestralMixin, DescendantMixin):
    """Mixin for two-way fan chart functionality."""
    
    def set_values(self, root_person_handle, maxgen_asc, maxgen_desc, 
                   flipupsidedownname, twolinename, background, 
                   background_gradient, fontdescr, grad_start, grad_end,
                   filtr, alpha_filter, angle_algo, dupcolor, showid):
        """Set values for two-way fan chart."""
        self.rootpersonh = root_person_handle
        self.config.generations_asc = maxgen_asc
        self.config.generations_desc = maxgen_desc
        self.config.flipupsidedownname = flipupsidedownname
        self.config.twolinename = twolinename
        self.config.background = background
        self.config.background_gradient = background_gradient
        self.config.font = fontdescr
        self.config.grad_start = grad_start
        self.config.grad_end = grad_end
        self.config.filter = filtr
        self.config.alpha_filter = alpha_filter
        self.config.angle_algo = angle_algo
        self.config.dupcolor = dupcolor
        self.config.showid = showid
        self._apply_config_to_properties()

# Updated widget classes using mixins
class FanChartWidget(AncestralMixin, FanChartBaseWidget):
    """Ancestral fan chart widget."""
    
    def __init__(self, dbstate, uistate, callback_popup=None):
        super().__init__(dbstate, uistate, callback_popup, CHART_ANCESTRAL)

class FanChartDescWidget(DescendantMixin, FanChartBaseWidget):
    """Descendant fan chart widget."""
    
    def __init__(self, dbstate, uistate, callback_popup=None):
        super().__init__(dbstate, uistate, callback_popup, CHART_DESCENDANT)

class FanChart2WayWidget(TwoWayMixin, FanChartBaseWidget):
    """Two-way fan chart widget."""
    
    def __init__(self, dbstate, uistate, callback_popup=None):
        super().__init__(dbstate, uistate, callback_popup, CHART_TWOWAY)
```

## Phase 3: Unified GUI Interface (Week 5-6)

### Step 3.1: Create Unified FanChartGrampsGUI

```python
class FanChartGrampsGUI:
    """
    Unified GUI interface for all fan chart types.
    This class provides common GUI functionality for fan charts,
    including menus, configuration management, and user interaction.
    """

    def __init__(self, callback_popup, chart_type=CHART_ANCESTRAL):
        """
        Initialize the GUI interface.
        
        :param callback_popup: Callback for popup menu
        :param chart_type: Type of fan chart
        """
        self.callback_popup = callback_popup
        self.chart_type = chart_type
        self.config = FanChartConfig(chart_type)
        self.fan = None
        self.dbstate = None
        self.uistate = None
        
        # Common UI elements
        self._setup_common_ui()
    
    def set_fan(self, fan_widget):
        """
        Set the fan chart widget.
        
        :param fan_widget: Fan chart widget instance
        """
        self.fan = fan_widget
        self.fan.set_config(self.config)
        self._connect_signals()
    
    def set_dbstate(self, dbstate):
        """Set the database state."""
        self.dbstate = dbstate
        if self.fan:
            self.fan.dbstate = dbstate
    
    def set_uistate(self, uistate):
        """Set the UI state."""
        self.uistate = uistate
        if self.fan:
            self.fan.uistate = uistate
    
    def _setup_common_ui(self):
        """Setup common UI elements."""
        # This will be implemented with common menu items
        pass
    
    def _connect_signals(self):
        """Connect common signals."""
        if self.fan:
            self.fan.connect("button-press-event", self.on_mouse_down)
            # ... other common signal connections
    
    def on_childmenu_changed(self, obj, person_handle):
        """
        Callback for the pulldown menu selection, changing to the person
        attached with menu item.
        """
        if self.callback_popup:
            return self.callback_popup(obj, person_handle)
        return False
    
    def update(self):
        """Update the fan chart display."""
        if self.fan:
            self.fan.reset()
            self.fan.queue_draw()
    
    def active_changed(self, handle):
        """
        Method called when active person changes.
        """
        # Reset everything but rotation angle (leave it as is)
        if self.fan:
            old_rotate = self.fan.rotate_value
            self.fan.reset()
            self.fan.rotate_value = old_rotate
            self.fan.queue_draw()
    
    def change_db(self, db):
        """
        Called when the database changes.
        """
        if self.fan:
            self.fan.dbstate = self.dbstate
            self.update()
    
    def on_popup(self, obj, event):
        """
        Handle popup menu for the fan chart.
        """
        # Common popup menu logic
        pass
    
    def build_menu(self):
        """
        Build the context menu for the fan chart.
        """
        menu = Gtk.Menu()
        
        # Add common menu items
        self._add_common_menu_items(menu)
        
        # Add type-specific menu items
        if self.chart_type == CHART_ANCESTRAL:
            self._add_ancestral_menu_items(menu)
        elif self.chart_type == CHART_DESCENDANT:
            self._add_descendant_menu_items(menu)
        elif self.chart_type == CHART_TWOWAY:
            self._add_twoway_menu_items(menu)
        
        return menu
```

### Step 3.2: Update Existing GUI Classes

Make the existing GUI classes inherit from the unified interface:

```python
# In fanchart.py
class FanChartGrampsGUI(FanChartGrampsGUIBase):
    """GUI interface for ancestral fan charts."""
    
    def __init__(self, callback_popup):
        super().__init__(callback_popup, CHART_ANCESTRAL)

# In fanchartdesc.py  
class FanChartDescGrampsGUI(FanChartGrampsGUIBase):
    """GUI interface for descendant fan charts."""
    
    def __init__(self, callback_popup):
        super().__init__(callback_popup, CHART_DESCENDANT)

# In fanchart2way.py
class FanChart2WayGrampsGUI(FanChartGrampsGUIBase):
    """GUI interface for two-way fan charts."""
    
    def __init__(self, callback_popup):
        super().__init__(callback_popup, CHART_TWOWAY)
```

## Phase 4: View and Gramplet Refactoring (Week 7-8)

### Step 4.1: Create Base View Class

```python
class BaseFanChartView(NavigationView):
    """
    Base class for all fan chart views.
    Provides common functionality and reduces code duplication.
    """

    # Common configuration settings for all fan chart views
    COMMON_CONFIGSETTINGS = (
        ("interface.fanview-background", BACKGROUND_GRAD_GEN),
        ("interface.fanview-radialtext", True),
        ("interface.fanview-twolinename", True),
        ("interface.fanview-flipupsidedownname", True),
        ("interface.fanview-font", "Sans"),
        ("interface.fanview-form", FORM_CIRCLE),
        ("interface.fanview-showid", False),
        ("interface.color-start-grad", "#ef2929"),
        ("interface.color-end-grad", "#3d37e9"),
    )

    def __init__(self, title, pdata, dbstate, uistate, nav_group=0, chart_type=CHART_ANCESTRAL):
        """
        Initialize the fan chart view.
        
        :param title: View title
        :param pdata: Plugin data
        :param dbstate: Database state
        :param uistate: UI state
        :param nav_group: Navigation group
        :param chart_type: Type of fan chart
        """
        self.chart_type = chart_type
        self.dbstate = dbstate
        self.uistate = uistate
        
        NavigationView.__init__(
            self, title, pdata, dbstate, uistate, PersonBookmarks, nav_group
        )
        
        # Import the appropriate GUI class based on chart type
        self._import_gui_class()
        
        # Initialize the GUI interface
        self.gui_interface = self.gui_class(self.on_childmenu_changed)
        
        # Set up configuration
        self._setup_configuration()
        
        # Set up common connections
        dbstate.connect("active-changed", self.active_changed)
        dbstate.connect("database-changed", self.change_db)
        
        # Add common UI
        self.additional_uis.append(self.additional_ui)
        self.allfonts = [x for x in enumerate(SystemFonts().get_system_fonts())]
        
        self.uistate.connect("font-changed", self.font_changed)
    
    def _import_gui_class(self):
        """Import the appropriate GUI class based on chart type."""
        if self.chart_type == CHART_ANCESTRAL:
            from gramps.gui.widgets.fanchart import FanChartGrampsGUI, FanChartWidget
            self.gui_class = FanChartGrampsGUI
            self.widget_class = FanChartWidget
        elif self.chart_type == CHART_DESCENDANT:
            from gramps.gui.widgets.fanchartdesc import FanChartDescGrampsGUI, FanChartDescWidget
            self.gui_class = FanChartDescGrampsGUI
            self.widget_class = FanChartDescWidget
        elif self.chart_type == CHART_TWOWAY:
            from gramps.gui.widgets.fanchart2way import FanChart2WayGrampsGUI, FanChart2WayWidget
            self.gui_class = FanChart2WayGrampsGUI
            self.widget_class = FanChart2WayWidget
    
    def _setup_configuration(self):
        """Setup configuration from config file."""
        scg = self._config.get
        
        # Common configuration
        self.background = scg("interface.fanview-background")
        self.radialtext = scg("interface.fanview-radialtext")
        self.twolinename = scg("interface.fanview-twolinename")
        self.flipupsidedownname = scg("interface.fanview-flipupsidedownname")
        self.fonttype = scg("interface.fanview-font")
        self.grad_start = scg("interface.color-start-grad")
        self.grad_end = scg("interface.color-end-grad")
        self.form = scg("interface.fanview-form")
        self.showid = scg("interface.fanview-showid")
        self.generic_filter = None
        self.alpha_filter = 0.2
        self.scrolledwindow = None
        
        # Type-specific configuration
        if self.chart_type == CHART_ANCESTRAL:
            self.maxgen = scg("interface.fanview-maxgen")
            self.childring = scg("interface.fanview-childrenring")
        elif self.chart_type == CHART_DESCENDANT:
            self.maxgen = scg("interface.fanview-maxgen")
            self.angle_algo = scg("interface.angle-algorithm")
            self.dupcolor = scg("interface.duplicate-color")
        elif self.chart_type == CHART_TWOWAY:
            self.generations_asc = scg("interface.fanview-maxgen-asc")
            self.generations_desc = scg("interface.fanview-maxgen-desc")
            self.background_gradient = scg("interface.fanview-background-gradient")
            self.angle_algo = scg("interface.angle-algorithm")
            self.dupcolor = scg("interface.duplicate-color")
    
    def build_widget(self):
        """Build the widget for the view."""
        # Create the appropriate widget
        fan_widget = self.widget_class(self.dbstate, self.uistate, self.on_popup)
        
        # Set the widget in the GUI interface
        self.gui_interface.set_fan(fan_widget)
        self.gui_interface.set_dbstate(self.dbstate)
        self.gui_interface.set_uistate(self.uistate)
        
        # Apply configuration to the widget
        self._apply_configuration_to_widget(fan_widget)
        
        self.scrolledwindow = Gtk.ScrolledWindow(hadjustment=None, vadjustment=None)
        self.scrolledwindow.set_policy(
            Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC
        )
        self.fan = fan_widget
        self.fan.show_all()
        self.scrolledwindow.add(self.fan)
        
        return self.scrolledwindow
    
    def _apply_configuration_to_widget(self, widget):
        """Apply configuration to the widget."""
        if self.chart_type == CHART_ANCESTRAL:
            widget.set_values(
                None,  # root_person_handle - will be set later
                self.maxgen,
                self.background,
                self.childring,
                self.flipupsidedownname,
                self.twolinename,
                self.radialtext,
                self.fonttype,
                self.grad_start,
                self.grad_end,
                self.generic_filter,
                self.alpha_filter,
                self.form,
                self.showid,
            )
        elif self.chart_type == CHART_DESCENDANT:
            widget.set_values(
                None,
                self.maxgen,
                self.flipupsidedownname,
                self.twolinename,
                self.background,
                self.fonttype,
                self.grad_start,
                self.grad_end,
                self.generic_filter,
                self.alpha_filter,
                self.form,
                self.angle_algo,
                self.dupcolor,
                self.showid,
            )
        elif self.chart_type == CHART_TWOWAY:
            widget.set_values(
                None,
                self.generations_asc,
                self.generations_desc,
                self.flipupsidedownname,
                self.twolinename,
                self.background,
                self.background_gradient,
                self.fonttype,
                self.grad_start,
                self.grad_end,
                self.generic_filter,
                self.alpha_filter,
                self.angle_algo,
                self.dupcolor,
                self.showid,
            )
    
    def font_changed(self):
        """Handle font changed event."""
        if hasattr(self.gui_interface, 'format_helper'):
            self.gui_interface.format_helper.reload_symbols()
        self.update()
    
    def navigation_type(self):
        """Return the navigation type."""
        return "Person"
    
    def get_handle_from_gramps_id(self, gid):
        """Return the handle of the specified object."""
        obj = self.dbstate.db.get_person_from_gramps_id(gid)
        if obj:
            return obj.get_handle()
        else:
            return None
    
    def get_stock(self):
        """Return the category stock icon."""
        return "gramps-pedigree"
    
    def get_viewtype_stock(self):
        """Return the type of view in category."""
        return "gramps-fanchart"
```

### Step 4.2: Update Existing Views

Simplify the existing views to inherit from the base class:

```python
# FanChartView becomes much simpler
class FanChartView(BaseFanChartView):
    """Fan Chart View for ancestral charts."""
    
    CONFIGSETTINGS = BaseFanChartView.COMMON_CONFIGSETTINGS + (
        ("interface.fanview-maxgen", 9),
        ("interface.fanview-childrenring", True),
    )

    def __init__(self, pdata, dbstate, uistate, nav_group=0):
        super().__init__(
            _("Fan Chart"), 
            pdata, 
            dbstate, 
            uistate, 
            nav_group,
            chart_type=CHART_ANCESTRAL
        )

# Similarly for other views
class FanChartDescView(BaseFanChartView):
    """Fan Chart View for descendant charts."""
    
    CONFIGSETTINGS = BaseFanChartView.COMMON_CONFIGSETTINGS + (
        ("interface.fanview-maxgen", 9),
        ("interface.angle-algorithm", ANGLE_WEIGHT),
        ("interface.duplicate-color", "#888a85"),
    )

    def __init__(self, pdata, dbstate, uistate, nav_group=0):
        super().__init__(
            _("Descendant Fan Chart"), 
            pdata, 
            dbstate, 
            uistate, 
            nav_group,
            chart_type=CHART_DESCENDANT
        )

class FanChart2WayView(BaseFanChartView):
    """Fan Chart View for two-way charts."""
    
    CONFIGSETTINGS = BaseFanChartView.COMMON_CONFIGSETTINGS + (
        ("interface.fanview-maxgen-asc", 4),
        ("interface.fanview-maxgen-desc", 4),
        ("interface.fanview-background-gradient", True),
        ("interface.angle-algorithm", ANGLE_WEIGHT),
        ("interface.duplicate-color", "#888a85"),
    )

    def __init__(self, pdata, dbstate, uistate, nav_group=0):
        super().__init__(
            _("2-Way Fan Chart"), 
            pdata, 
            dbstate, 
            uistate, 
            nav_group,
            chart_type=CHART_TWOWAY
        )
```

### Step 4.3: Create Base Gramplet Class

```python
class BaseFanChartGramplet(Gramplet):
    """
    Base class for all fan chart gramplets.
    """

    def __init__(self, gui, nav_group=0, chart_type=CHART_ANCESTRAL):
        """
        Initialize the fan chart gramplet.
        
        :param gui: GUI interface
        :param nav_group: Navigation group
        :param chart_type: Type of fan chart
        """
        Gramplet.__init__(self, gui, nav_group)
        self.chart_type = chart_type
        
        # Import the appropriate classes
        self._import_classes()
        
        # Initialize the GUI interface
        self.gui_interface = self.gui_class(self.on_childmenu_changed)
        self.gui_interface.set_dbstate(self.dbstate)
        self.gui_interface.set_uistate(self.uistate)
        
        # Create the widget
        self.fan = self.widget_class(self.dbstate, self.uistate, self.on_popup)
        self.gui_interface.set_fan(self.fan)
        
        # Apply default configuration
        self._apply_default_configuration()
        
        # Replace the standard textview with the fan chart widget
        self.gui.get_container_widget().remove(self.gui.textview)
        self.gui.get_container_widget().add(self.fan)
        self.fan.show()
    
    def _import_classes(self):
        """Import the appropriate classes based on chart type."""
        if self.chart_type == CHART_ANCESTRAL:
            from gramps.gui.widgets.fanchart import FanChartGrampsGUI, FanChartWidget
            self.gui_class = FanChartGrampsGUI
            self.widget_class = FanChartWidget
        elif self.chart_type == CHART_DESCENDANT:
            from gramps.gui.widgets.fanchartdesc import FanChartDescGrampsGUI, FanChartDescWidget
            self.gui_class = FanChartDescGrampsGUI
            self.widget_class = FanChartDescWidget
        elif self.chart_type == CHART_TWOWAY:
            from gramps.gui.widgets.fanchart2way import FanChart2WayGrampsGUI, FanChart2WayWidget
            self.gui_class = FanChart2WayGrampsGUI
            self.widget_class = FanChart2WayWidget
    
    def _apply_default_configuration(self):
        """Apply default configuration to the widget."""
        if self.chart_type == CHART_ANCESTRAL:
            self.fan.set_values(
                None, 6, BACKGROUND_SCHEME1, True, True, True, True,
                "Sans", "#0000FF", "#FF0000", None, 0.2, FORM_HALFCIRCLE, False
            )
        elif self.chart_type == CHART_DESCENDANT:
            self.fan.set_values(
                None, 6, True, True, BACKGROUND_SCHEME1, "Sans",
                "#0000FF", "#FF0000", None, 0.2, FORM_HALFCIRCLE,
                ANGLE_WEIGHT, "#888A85", False
            )
        elif self.chart_type == CHART_TWOWAY:
            self.fan.set_values(
                None, 5, 4, True, True, BACKGROUND_SCHEME1, True,
                "Sans", "#FF0000", "#0000FF", None, 0.2,
                ANGLE_WEIGHT, "#888A85", False
            )
    
    def init(self):
        """Initialize the gramplet."""
        self.set_tooltip(
            _(
                "Click to expand/contract person\n"
                "Right-click for options\n"
                "Click and drag in open area to rotate"
            )
        )
    
    def active_changed(self, handle):
        """Method called when active person changes."""
        # Reset everything but rotation angle (leave it as is)
        if self.gui_interface:
            self.gui_interface.active_changed(handle)
    
    def on_childmenu_changed(self, obj, person_handle):
        """Callback for the pulldown menu selection."""
        self.set_active("Person", person_handle)
        return True
```

### Step 4.4: Update Existing Gramplets

```python
# FanChartGramplet becomes much simpler
class FanChartGramplet(BaseFanChartGramplet):
    """Fan Chart Gramplet for ancestral charts."""
    
    def __init__(self, gui, nav_group=0):
        super().__init__(gui, nav_group, CHART_ANCESTRAL)

# Similarly for other gramplets
class FanChartDescGramplet(BaseFanChartGramplet):
    """Fan Chart Gramplet for descendant charts."""
    
    def __init__(self, gui, nav_group=0):
        super().__init__(gui, nav_group, CHART_DESCENDANT)

class FanChart2WayGramplet(BaseFanChartGramplet):
    """Fan Chart Gramplet for two-way charts."""
    
    def __init__(self, gui, nav_group=0):
        super().__init__(gui, nav_group, CHART_TWOWAY)
```

## Phase 5: Report Extension (Week 9-10)

### Step 5.1: Extend FanChart Report

```python
class FanChartReport(Report):
    """
    Unified fan chart report supporting all chart types.
    """

    def __init__(self, database, options, user):
        """
        Create the FanChart object that produces the report.
        """
        Report.__init__(self, database, options, user)
        
        menu = options.menu
        self.set_locale(options.menu.get_option_by_name("trans").get_value())

        stdoptions.run_private_data_option(self, menu)
        stdoptions.run_living_people_option(self, menu, self._locale)
        self.database = CacheProxyDb(self.database)

        # Determine chart type
        self.chart_type = menu.get_option_by_name("chart_type").get_value()
        
        # Common options
        self.max_generations = menu.get_option_by_name("maxgen").get_value()
        self.circle = menu.get_option_by_name("circle").get_value()
        self.background = menu.get_option_by_name("background").get_value()
        self.radial = menu.get_option_by_name("radial").get_value()
        pid = menu.get_option_by_name("pid").get_value()
        self.draw_empty = menu.get_option_by_name("draw_empty").get_value()
        self.same_style = menu.get_option_by_name("same_style").get_value()
        self.flip_text = menu.get_option_by_name("flip_text").get_value()
        self.exclude_title = menu.get_option_by_name("exclude_title").get_value()
        
        self.center_person = self.database.get_person_from_gramps_id(pid)
        if self.center_person is None:
            raise ReportError(_("Person %s is not in the Database") % pid)

        # Type-specific options
        if self.chart_type == CHART_DESCENDANT:
            self.angle_algo = menu.get_option_by_name("angle_algo").get_value()
            self.dupcolor = menu.get_option_by_name("dupcolor").get_value()
        elif self.chart_type == CHART_TWOWAY:
            self.generations_asc = menu.get_option_by_name("maxgen_asc").get_value()
            self.generations_desc = menu.get_option_by_name("maxgen_desc").get_value()
            self.background_gradient = menu.get_option_by_name("background_gradient").get_value()
        
        # Setup styles
        self.graphic_style = []
        self.text_style = []
        for i in range(0, self.max_generations):
            self.graphic_style.append("FC-Graphic" + "%02d" % i)
            self.text_style.append("FC-Text" + "%02d" % i)

        self.calendar = 0
        self.height = 0
        self.map = [None] * 2**self.max_generations
        self.text = {}
    
    def write_report(self):
        """Write the fan chart report."""
        self.doc.start_page()

        if self.chart_type == CHART_ANCESTRAL:
            self._write_ancestral_report()
        elif self.chart_type == CHART_DESCENDANT:
            self._write_descendant_report()
        elif self.chart_type == CHART_TWOWAY:
            self._write_twoway_report()
        
        self.doc.end_page()
    
    def _write_ancestral_report(self):
        """Write ancestral fan chart report."""
        # Existing ancestral logic
        self.apply_filter(self.center_person.get_handle(), 1)
        p_rn = self.center_person.get_primary_name().get_regular_name()
        # ... rest of existing ancestral report logic
    
    def _write_descendant_report(self):
        """Write descendant fan chart report."""
        # New descendant report logic
        pass
    
    def _write_twoway_report(self):
        """Write two-way fan chart report."""
        # New two-way report logic
        pass
```

### Step 5.2: Add Report Options

```python
# In the report's options
class FanChartOptions(MenuReportOptions):
    """
    Options for the Fan Chart report.
    """

    def __init__(self, name, dbase):
        MenuReportOptions.__init__(self, name, dbase)

    def add_menu_options(self, menu):
        """Add menu options for the report."""
        # Chart type option
        chart_type_option = EnumeratedListOption(
            _("Chart Type"),
            "chart_type",
            [
                (CHART_ANCESTRAL, _("Ancestral")),
                (CHART_DESCENDANT, _("Descendant")),
                (CHART_TWOWAY, _("Two-way")),
            ],
            CHART_ANCESTRAL
        )
        menu.add_option(chart_type_option)
        
        # Common options
        maxgen_option = NumberOption(
            _("Maximum generations"),
            "maxgen",
            9,
            1, 15
        )
        menu.add_option(maxgen_option)
        
        # Add other common options...
        
        # Type-specific options will be added based on chart type
        # This can be done in the report's __init__ method
```

## Phase 6: Testing and Validation (Week 11-12)

### Step 6.1: Unit Tests

Create comprehensive unit tests for:
- Configuration management
- Widget rendering
- Data traversal logic
- User interaction
- Report generation

### Step 6.2: Integration Tests

Test the integration between:
- Views and widgets
- Gramplets and widgets
- Reports and database
- Configuration persistence

### Step 6.3: Regression Tests

Ensure all existing functionality continues to work:
- All existing fan chart types
- All existing configuration options
- All existing user interactions

### Step 6.4: User Testing

Get feedback from users on:
- New unified interface
- Consistent options across chart types
- Extended report functionality

## Implementation Timeline

| Phase | Duration | Deliverables |
|-------|----------|--------------|
| 1. Configuration | 2 weeks | Unified config module, updated constants |
| 2. Base Widget | 2 weeks | Enhanced base widget, type-specific mixins |
| 3. GUI Interface | 2 weeks | Unified GUI interface, updated existing classes |
| 4. Views/Gramplets | 2 weeks | Base classes, simplified existing implementations |
| 5. Reports | 2 weeks | Extended report with all chart types |
| 6. Testing | 2 weeks | Comprehensive test suite |

**Total**: 12 weeks

## Risk Assessment

### Low Risk Items:
- Configuration consolidation
- Base widget enhancement
- GUI interface unification

### Medium Risk Items:
- View and gramplet refactoring (affects user experience)
- Report extension (new functionality)

### High Risk Items:
- Multiple inheritance resolution (current FanChart2WayWidget uses multiple inheritance)
- Backward compatibility maintenance

## Mitigation Strategies

1. **Incremental Implementation**: Implement one phase at a time with thorough testing
2. **Backward Compatibility**: Maintain all existing APIs and configuration keys
3. **Feature Flags**: Use feature flags to enable/disable new functionality
4. **User Feedback**: Get early feedback on each phase
5. **Rollback Plan**: Maintain ability to rollback to previous implementation

## Success Criteria

1. **Code Reduction**: At least 30% reduction in total lines of code for fan chart implementations
2. **Feature Parity**: All existing features available in all chart types where applicable
3. **Extended Reports**: Descendant and two-way fan chart reports available
4. **Consistent UI**: Unified configuration and options across all chart types
5. **Backward Compatibility**: All existing configurations and usage patterns continue to work
6. **Test Coverage**: 100% test coverage for new consolidated code

## Next Steps

1. **Review and Approval**: Get approval on this implementation plan
2. **Setup Development Environment**: Create feature branch for development
3. **Phase 1 Implementation**: Start with configuration consolidation
4. **Iterative Development**: Implement each phase with testing and review

This plan addresses the original request to "Consolidate Fan charts types and features to extend Report choices" while significantly improving the codebase architecture and maintainability.
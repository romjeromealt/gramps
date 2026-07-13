# Fan Chart Consolidation Analysis

## Executive Summary

This document analyzes the current Fan Chart implementations across Gramps and proposes a consolidation strategy to reduce code duplication and provide consistent features across all Fan Chart types.

## Current State Analysis

### 1. Widget Architecture (gramps/gui/widgets/)

#### Base Class: FanChartBaseWidget
- **File**: `fanchart.py` (2467 lines)
- **Purpose**: Common functionality for all fan chart widgets
- **Key Features**:
  - GTK DrawingArea base
  - Mouse interaction (click, drag, rotate)
  - Drawing and rendering logic
  - Configuration management
  - Common constants and utilities

#### Specialized Widgets:

1. **FanChartWidget** (Ancestral)
   - **File**: `fanchart.py` (part of same file)
   - **Inheritance**: FanChartBaseWidget
   - **Purpose**: Ancestral fan charts (traditional pedigree)
   - **Unique Features**: 
     - Ancestor traversal logic
     - Single root person with ancestors
     - `set_values()` with ancestral-specific parameters

2. **FanChartDescWidget** (Descendant)
   - **File**: `fanchartdesc.py` (820 lines)
   - **Inheritance**: FanChartBaseWidget
   - **Purpose**: Descendant fan charts
   - **Unique Features**:
     - Descendant traversal logic
     - Multiple people per generation (families)
     - Angle calculation algorithms (ANGLE_CHEQUI, ANGLE_WEIGHT)
     - Family/partner handling
     - Duplicate person detection

3. **FanChart2WayWidget** (Two-way)
   - **File**: `fanchart2way.py` (836 lines)
   - **Inheritance**: FanChartWidget + FanChartDescWidget (multiple inheritance)
   - **Purpose**: Combined ancestral and descendant fan charts
   - **Unique Features**:
     - Separate generation limits for ancestors and descendants
     - Combined data structures from both parent classes
     - Background gradient option for distinguishing asc/desc

### 2. View Plugins (gramps/plugins/view/)

Each widget type has a corresponding view plugin:

- **FanChartView**: Uses FanChartWidget
- **FanChartDescView**: Uses FanChartDescWidget  
- **FanChart2WayView**: Uses FanChart2WayWidget

**Common Pattern**: All inherit from both their respective GUI class and `NavigationView`

### 3. Gramplets (gramps/plugins/gramplet/)

Each widget type has a corresponding gramplet:

- **FanChartGramplet**: Uses FanChartWidget
- **FanChartDescGramplet**: Uses FanChartDescWidget
- **FanChart2WayGramplet**: Uses FanChart2WayWidget

**Common Pattern**: All inherit from both their respective GUI class and `Gramplet`

### 4. Reports (gramps/plugins/drawreport/)

- **FanChart**: Traditional ancestral fan chart report
- **D3 Ancestral Fan**: Addon (not in core)

## Feature Comparison Matrix

### Common Features Across All Types:
| Feature | FanChartWidget | FanChartDescWidget | FanChart2WayWidget | FanChartReport |
|---------|----------------|-------------------|-------------------|----------------|
| Basic rendering | ✓ | ✓ | ✓ | ✓ |
| Mouse interaction | ✓ | ✓ | ✓ | ✗ |
| Rotation | ✓ | ✓ | ✓ | ✗ |
| Zoom/Translation | ✓ | ✓ | ✓ | ✗ |
| Color schemes | ✓ | ✓ | ✓ | Limited |
| Form types (circle/half/quadrant) | ✓ | ✓ | ✓ | ✓ |
| Background options | ✓ | ✓ | ✓ | Limited |
| Name display options | ✓ | ✓ | ✓ | ✓ |
| Font customization | ✓ | ✓ | ✓ | ✓ |
| ID display | ✓ | ✓ | ✓ | ✗ |
| Filter support | ✓ | ✓ | ✓ | ✗ |

### Unique Features:

**FanChartWidget (Ancestral)**:
- Simple ancestor-only traversal
- Single person per position
- Children ring option

**FanChartDescWidget (Descendant)**:
- Descendant traversal with families
- Multiple people per generation
- Angle calculation algorithms (ANGLE_CHEQUI, ANGLE_WEIGHT)
- Partner/family visualization
- Duplicate person handling
- Family ring display

**FanChart2WayWidget (Two-way)**:
- Combined ancestral and descendant
- Separate generation limits for each direction
- Background gradient for asc/desc distinction
- Complex data structure combining both types

**FanChartReport**:
- PDF/SVG output
- Overhang option (unique to report)
- Radial text options
- Print-optimized layout

## Code Duplication Issues

### 1. Configuration Management
Each view and gramplet duplicates configuration settings with slight variations:

```python
# FanChartView CONFIGSETTINGS
CONFIGSETTINGS = (
    ("interface.fanview-maxgen", 9),
    ("interface.fanview-background", fanchart.BACKGROUND_GRAD_GEN),
    ("interface.fanview-childrenring", True),
    ("interface.fanview-radialtext", True),
    ("interface.fanview-twolinename", True),
    ("interface.fanview-flipupsidedownname", True),
    ("interface.fanview-font", "Sans"),
    ("interface.fanview-form", fanchart.FORM_CIRCLE),
    ("interface.fanview-showid", False),
    ("interface.color-start-grad", "#ef2929"),
    ("interface.color-end-grad", "#3d37e9"),
)

# FanChartDescView CONFIGSETTINGS
CONFIGSETTINGS = (
    ("interface.fanview-maxgen", 9),
    ("interface.fanview-background", fanchart.BACKGROUND_GRAD_GEN),
    ("interface.fanview-radialtext", True),
    ("interface.fanview-twolinename", True),
    ("interface.fanview-flipupsidedownname", True),
    ("interface.fanview-font", "Sans"),
    ("interface.fanview-form", fanchart.FORM_CIRCLE),
    ("interface.angle-algorithm", fanchartdesc.ANGLE_WEIGHT),
    ("interface.duplicate-color", "#888a85"),
    ("interface.fanview-showid", False),
)
```

### 2. GUI Classes
Each widget type has its own GUI class (FanChartGrampsGUI, FanChartDescGrampsGUI, FanChart2WayGrampsGUI) with duplicated menu and configuration logic.

### 3. View and Gramplet Initialization
Each view and gramplet duplicates the initialization pattern:
- Setting up configuration
- Creating the widget
- Connecting signals
- Setting up UI elements

## Proposed Consolidation Architecture

### 1. Unified Configuration System

Create a centralized configuration manager for fan charts:

```python
class FanChartConfig:
    """Unified configuration for all fan chart types"""
    
    # Common settings
    COMMON_SETTINGS = {
        'maxgen': 9,
        'background': BACKGROUND_GRAD_GEN,
        'radialtext': True,
        'twolinename': True,
        'flipupsidedownname': True,
        'font': "Sans",
        'form': FORM_CIRCLE,
        'showid': False,
        'color_start_grad': "#ef2929",
        'color_end_grad': "#3d37e9",
        'alpha_filter': 0.2,
    }
    
    # Ancestral-specific settings
    ANCESTRAL_SETTINGS = {
        'childrenring': True,
    }
    
    # Descendant-specific settings  
    DESCENDANT_SETTINGS = {
        'angle_algorithm': ANGLE_WEIGHT,
        'duplicate_color': "#888a85",
    }
    
    # Two-way specific settings
    TWOWAY_SETTINGS = {
        'maxgen_asc': 4,
        'maxgen_desc': 4,
        'background_gradient': True,
    }
```

### 2. Unified Widget Base Class

Enhance `FanChartBaseWidget` to handle all fan chart types with configuration:

```python
class FanChartBaseWidget(Gtk.DrawingArea):
    """Unified base widget for all fan chart types"""
    
    def __init__(self, dbstate, uistate, callback_popup=None, chart_type='ancestral'):
        # Common initialization
        self.chart_type = chart_type  # 'ancestral', 'descendant', 'twoway'
        self.common_init(dbstate, uistate, callback_popup)
        
        # Type-specific initialization
        if chart_type == 'ancestral':
            self.ancestral_init()
        elif chart_type == 'descendant':
            self.descendant_init()
        elif chart_type == 'twoway':
            self.twoway_init()
```

### 3. Unified GUI Interface

Create a single `FanChartGrampsGUI` class that can handle all chart types:

```python
class FanChartGrampsGUI:
    """Unified GUI interface for all fan chart types"""
    
    def __init__(self, callback_popup, chart_type='ancestral'):
        self.chart_type = chart_type
        self.callback_popup = callback_popup
        self.setup_common_ui()
        self.setup_type_specific_ui()
    
    def set_fan(self, fan_widget):
        """Set the fan chart widget"""
        self.fan = fan_widget
        self.fan.set_config(self.config)
```

### 4. Generic View and Gramplet Base Classes

Create base classes that handle common functionality:

```python
class BaseFanChartView(NavigationView):
    """Base class for all fan chart views"""
    
    def __init__(self, title, pdata, dbstate, uistate, nav_group=0, chart_type='ancestral'):
        self.chart_type = chart_type
        NavigationView.__init__(self, title, pdata, dbstate, uistate, PersonBookmarks, nav_group)
        self.setup_fan_chart()
    
    def setup_fan_chart(self):
        """Setup the appropriate fan chart widget based on type"""
        if self.chart_type == 'ancestral':
            from gramps.gui.widgets.fanchart import FanChartWidget, FanChartGrampsGUI
            self.gui_class = FanChartGrampsGUI
            self.widget_class = FanChartWidget
        # ... etc for other types

class BaseFanChartGramplet(Gramplet):
    """Base class for all fan chart gramplets"""
    
    def __init__(self, gui, nav_group=0, chart_type='ancestral'):
        Gramplet.__init__(self, gui, nav_group)
        self.chart_type = chart_type
        self.setup_fan_chart()
```

### 5. Report Integration

Extend the report system to support all fan chart types:

```python
class FanChartReport(Report):
    """Unified fan chart report supporting all types"""
    
    def __init__(self, database, options, user):
        Report.__init__(self, database, options, user)
        self.chart_type = options.menu.get_option_by_name('chart_type').get_value()
        self.setup_chart_specific_options()
```

## Implementation Strategy

### Phase 1: Configuration Consolidation
1. Create unified configuration constants
2. Refactor existing classes to use common configuration
3. Ensure backward compatibility

### Phase 2: Widget Architecture Refactoring
1. Enhance `FanChartBaseWidget` with common functionality
2. Create type-specific mixins for unique features
3. Unify the widget hierarchy

### Phase 3: GUI Interface Consolidation
1. Create unified `FanChartGrampsGUI` class
2. Migrate existing GUI classes to use the unified interface
3. Standardize menu and UI elements

### Phase 4: View and Gramplet Refactoring
1. Create base classes for views and gramplets
2. Migrate existing implementations to use base classes
3. Ensure consistent behavior across all types

### Phase 5: Report Enhancement
1. Extend report to support descendant and two-way charts
2. Add consistent options across all chart types
3. Unify output formats

## Benefits of Consolidation

### 1. Reduced Code Duplication
- Eliminate duplicated configuration management
- Share common rendering and interaction logic
- Reduce maintenance burden

### 2. Consistent User Experience
- Same options available across all chart types where applicable
- Consistent behavior and appearance
- Unified configuration system

### 3. Extended Report Choices
- Add descendant and two-way fan chart reports
- Consistent options between screen display and reports
- Better feature parity

### 4. Easier Maintenance
- Fixes apply to all chart types simultaneously
- New features can be added to all types more easily
- Reduced testing surface area

### 5. Better Architecture
- Clear separation of concerns
- Type-specific functionality through composition
- Extensible for new chart types

## Migration Path

### Backward Compatibility
- Maintain existing configuration keys
- Preserve existing API for plugins
- Gradual migration with fallbacks

### Testing Strategy
- Comprehensive unit tests for each chart type
- Integration tests for views and gramplets
- Regression tests for existing functionality

### Documentation Updates
- Update user documentation for new unified options
- Create developer documentation for the new architecture
- Update wiki pages for all fan chart types

## Recommendations

1. **Start with Configuration**: The configuration consolidation will provide immediate benefits with minimal risk.

2. **Incremental Refactoring**: Refactor one component at a time, ensuring each change maintains backward compatibility.

3. **Feature Parity**: Ensure all chart types have access to the same features where applicable (e.g., all should support the same background options).

4. **Report Extension**: Prioritize adding descendant and two-way fan chart reports to address the original request.

5. **User Testing**: Get feedback from users on the consolidated interface to ensure it meets their needs.

This consolidation will significantly improve the codebase maintainability while extending the report choices as requested in issue 0012927.
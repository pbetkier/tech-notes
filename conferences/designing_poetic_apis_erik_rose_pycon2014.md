# Designing Poetic APIs - Erik Rose (PyCon 2014, Montreal)

Scientific fact: languague limits the way you think. So API design matters a lot. Finding good abstractions is important - you impact how people will think.

Use abstractions that already exist in the mental models of developers (urlib vs requests).

## 7 rules

### Extraction over invention
The best libraries are extracted from existing, successful code, not invented.

### Consistency
Be consistent with what is used in the industry already, if you have to innovate, at least be consistent with yourself!

### Brevity
Pay only for what you need. Make smart defaults.

### Composability
GNU/Linux tools style. Saves documentation length, limits test cases to write, etc. Watch out for deep inheritance hierarchies and classes with lots of state. If you have to mock a lot, you probably have to split your functionality.

### Plain data
Simple API uses basic data types and structures. Easier to use. API should expect only what it really needs.

### Grooviness
Feels good to use. Fail shallowly to make the traceback small.

### Safety
Dangerous operations should be required to be made explicitly. If your docs contain warnings, then you have a problem. Users will blame you, not themselves e.g. when they accidentally delete all their data.

More: a chapter in *Making Software: What Really Works, and Why We Believe It* by Andy Oram and Greg Wilson.


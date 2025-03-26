Derived from [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html) and adapted to our needs. Code standard encourages maintaining code consistency in several aspects: naming, format, structure, and documentation. Consistent code among many developers contributes to better readability, maintainability, and collaboration. We strongly recommend (if not enforce) to follow these guidelines. This document covers the visible part, the face, _“the skin“_ of our [Code Culture](Overview).

### Source file
- source file name consists of the case-sensitive name of the top-level class it contains
- source files are encoded in UTF-8
- any special escape sequence is preferred before the corresponding octal (`\012`) or Unicode (`\u000a`) escape
- for remaining non-ASCII characters, preferably use the actual Unicode character (e.g.`∞`) or Unicode escape (e.g. `\u221e`)
- source file consists of, **in order**:
   - package statement
   - import statements
   - exactly **one** top-level class
- the **package** statement **is not line-wrapped**
- the **import** statements **are not line-wrapped**
- if possible, avoid wildcard import statements (e.g. `org.junit.*`) [Why is using a wild card with a Java import statement bad?](https://stackoverflow.com/questions/147454/why-is-using-a-wild-card-with-a-java-import-statement-bad)
- imports are declared as follows, **in order**:
   - all non-static imports in a single block
   - all static imports in a single block
- if both _static_ and _non-static_ import blocks are present, **a single blank line separates them**. There are no other blank lines between import statements
- imported names appear in ASCII sort order within each block (this is not the same as import _statements_ being in ASCII sort order)
- static import is not used for static nested classes; they are imported via normal imports
- there is exactly **one** top-level class per source file of the same name
- the order of the class members is not set, **BUT** the class must use **some logical order** of its members.
### Formatting
- Checkstyle can be utilized to support applying code format
- It is recommended to import code format specification into your IDE
- Code format specification (one of many possible):
```XML
<code_scheme name="experimental" version="173">
	<JavaCodeStyleSettings>
		<option name="CLASS_COUNT_TO_USE_IMPORT_ON_DEMAND" value="99" />
		<option name="NAMES_COUNT_TO_USE_IMPORT_ON_DEMAND" value="99" />
	</JavaCodeStyleSettings>
	<codeStyleSettings language="JAVA">
		<option name="KEEP_LINE_BREAKS" value="false" />
		<option name="KEEP_CONTROL_STATEMENT_IN_ONE_LINE" value="false" />
		<option name="KEEP_BLANK_LINES_IN_CODE" value="1" />
		<option name="BLANK_LINES_AFTER_CLASS_HEADER" value="1" />
		<option name="ALIGN_MULTILINE_PARAMETERS" value="false" />
		<option name="ALIGN_MULTILINE_RESOURCES" value="false" />
		<option name="ALIGN_MULTILINE_FOR" value="false" />
		<option name="CALL_PARAMETERS_WRAP" value="1" />
		<option name="METHOD_PARAMETERS_WRAP" value="1" />
		<option name="EXTENDS_LIST_WRAP" value="1" />
		<option name="THROWS_KEYWORD_WRAP" value="1" />
		<option name="METHOD_CALL_CHAIN_WRAP" value="5" />
		<option name="WRAP_FIRST_METHOD_IN_CALL_CHAIN" value="true" />
		<option name="BINARY_OPERATION_WRAP" value="1" />
		<option name="BINARY_OPERATION_SIGN_ON_NEXT_LINE" value="true" />
		<option name="TERNARY_OPERATION_WRAP" value="1" />
		<option name="TERNARY_OPERATION_SIGNS_ON_NEXT_LINE" value="true" />
		<option name="FOR_STATEMENT_WRAP" value="1" />
		<option name="ARRAY_INITIALIZER_WRAP" value="1" />
		<option name="WRAP_COMMENTS" value="true" />
		<option name="IF_BRACE_FORCE" value="3" />
		<option name="DOWHILE_BRACE_FORCE" value="3" />
		<option name="WHILE_BRACE_FORCE" value="3" />
		<option name="FOR_BRACE_FORCE" value="3" />
	</codeStyleSettings>
	<XML>
		<option name="XML_ATTRIBUTE_WRAP" value="2" />
	</XML>
	<codeStyleSettings language="Impex">
		<option name="KEEP_BLANK_LINES_IN_CODE" value="1" />
	</codeStyleSettings>
	<codeStyleSettings language="XML">
		<option name="SOFT_MARGINS" value="80,120" />
		<indentOptions>
			<option name="USE_TAB_CHARACTER" value="true" />
			<option name="SMART_TABS" value="true" />
		</indentOptions>
	</codeStyleSettings>
	<JSCodeStyleSettings version="0">
		<option name="SPACE_BEFORE_FUNCTION_LEFT_PARENTH" value="false" />
	</JSCodeStyleSettings>
</code_scheme>
```
### Naming conventions
- use only ASCII letters, digits, and in some cases underscores for identifiers
- special prefixes/suffixes are discouraged
- package names use **only** lowercase letters and digits. Consecutive words are simply concatenated together. E.g. `com.example.deepspace`, not `com.example.deepSpace` or `com.example.deep_space`
- class/interface/enum class names are written in UpperCamelCase
- class/enum class names are typically nouns or noun phrases (e.g. `Character` or `ImmutableList`)
- interface names **may** also be adjectives or adjective phrases instead (e.g.`Readable`)
- test class has a name of the class it intends to test and ends with `Test` suffix (e.g. `HashIntegrationTest` testing class `HashIntegration`)
- method names are written in lowerCamelCase and are typically verbs of verb phrases (e.g. `sendMessage` or `stop`)
- constant names are written in UPPER_SNAKE_CASE and are typically nouns or noun phrases
- non-constant field names (static or not) are written in lowerCamelCase and are typically nouns or noun phrases (e.g. `computedValues` or `index`)
- parameter names are written in lowerCamelCase
- local variable names are written in lowerCamelCase
- local variables, even when final and immutable, are not considered to be constants and should not be styled as such
- type variable names are **either** a single capital letter (optionally followed by a single numeral) (e.g. `E`, `T`, `X`, `T2`) **OR** in the form used for classes followed by the capital `T` (e.g. `RequestT`, `FooBarT`)
- custom exception class ends with an `Exception` suffix
## Javadoc & Comments
documentation in its basic form should look something like this:

```java
/**
 * Multiple lines of Javadoc text are written here,
 * wrapped normally...
 */
public int method(String p1) { ... }
```

or
```java
/** An especially short bit of Javadoc. */
public int method(String p1) { ... }
```    

- Any of the standard "block tags" that are used appear in the order `@param`, `@return`, `@throws`, `@deprecated`, and these four types never appear with an empty description. When a block tag doesn't fit on a single line, continuation lines are indented four (or more) spaces from the position of the `@`.
- Each Javadoc block begins with a brief **summary fragment**. This fragment is crucial: it is the only part of the text that appears in certain contexts such as class and method indexes.
- At the _minimum_, Javadoc is present for every `public` class, and every `public` or `protected` member of such a class **EXCEPT** for “simple, obvious” members and overridden members
- While comments are important, the code should ideally be self-explanatory; **avoid comments that explain the obvious**. Clean code with readable identifier names should not require comments, **EXCEPT** when documenting operational assumptions, algorithm insights, and so on
- update comments on every change in the logic of a method/algorithm
- be consistent with the commenting style
- comment on every piece of code that **is not obvious** “at first sight”. Explain it from a rather high-level perspective. Provide a comment that increases readability and saves future developers time by providing a simple, straightforward idea of how the code works. Focus on the _‘why it is done’_ before _‘what is done’_. The code already documents _'what is done'_ (of course if it adheres to the code culture). Take this code:
```java
public static int[][] foo(int[][] A, int[][] B) { 
	int[][] result = new int[2][2];
	for (int i = 0; i < 2; i++) {
		for (int j = 0; j < 2; j++) {
			result[i][j] = 0;
			for (int k = 0; k < 2; k++) {
				result[i][j] += A[i][k] * B[k][j];
			}
		}
	}
	return result;
}
```

It is quite unclear what this piece of code does. However, after some non negligible time spent studying it, one can conclude that this method computes the multiplication of two matrices. Now consider this piece of code:

```java
/**
  * Multiplies two 2x2 matrices.
  * 
  * @param A The first 2x2 matrix.
  * @param B The second 2x2 matrix.
  * @return The product matrix of multiplying A by B.
  */
public static int[][] multiplyMatrices(int[][] A, int[][] B) {
	int[][] result = new int[2][2];
	for (int i = 0; i < 2; i++) {
		for (int j = 0; j < 2; j++) {
			result[i][j] = 0; // Initialize the result element
			for (int k = 0; k < 2; k++) {
				// Sum the product of the corresponding elements
				result[i][j] += A[i][k] * B[k][j];
			}
		}
	}
	return result;
}
```

Nice and easy to understand in a few seconds vs. a couple of minutes in the previous example
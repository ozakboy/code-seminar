# C# 程式碼撰寫規範 (C# Coding Guideline) 2.0

本文件旨在提供一個全面的 C# 程式碼撰寫指南，以便開發團隊能夠編寫出一致且易於維護的程式碼。遵循這些準則將有助於提高程式碼的品質，並確保整個專案的協作效率。
 
## 目錄
1. [通則](#1-通則)
2. [程式碼結構](#2-程式碼結構)
3. [程式碼樣式](#3-程式碼樣式)
4. [格式化規則](#4-格式化規則)
5. [命名規則](#5-命名規則)
6. [版本控制指南](#6-版本控制指南)
7. [文件化規範](#7-文件化規範)
8. [錯誤處理準則](#8-錯誤處理準則)
9. [其他](#9-其他)

## 1. 通則

* **縮排**：使用**空格**進行縮排，每個縮排層級為 **4 個空格**。

  這表示每一層的程式碼區塊都應該向右縮進 4 個空格，以增加程式碼的可讀性。

* **字元編碼**：所有檔案應使用 UTF-8-BOM 字元編碼。

  這能確保所有檔案 (包含程式碼、文字檔、設定檔等) 都能正確顯示各種字元，特別是中文等非 ASCII 字元。另外，含有「中文命名」的專案，這條規範尤其重要。

* **檔案結尾換行**：檔案結尾**不**需要插入新的一行。

  這是一種常見的程式碼規範，避免在檔案末尾出現多餘的空行。

## 2. 程式碼結構

### 2.1. using 指示詞

* `using` 指示詞應根據以下規則排序：
  * 系統命名空間 (例如 `System.*`) 應排在前面。

    這能幫助開發者快速識別程式碼使用的系統相關資源。

  * 不同的 `using` 指示詞群組應以空白行分隔。

    透過分組和空白行區隔，可以更清晰地了解程式碼的依賴關係。

* `using` 指示詞應放在命名空間之外。

  這確保了 `using` 指示詞的作用範圍涵蓋整個檔案，並且符合常見的程式碼風格。

* `using` 宣告應使用簡化的 `using` 語句形式 (例如 `using var foo = ...;`)。

  這種簡化的語法能使程式碼更簡潔，並減少程式碼的冗餘。

### 2.2. 命名空間

* 命名空間應使用檔案範圍的命名空間宣告 (file-scoped namespace) (例如 `namespace MyNamespace;`)。

  這種宣告方式更簡潔，並且能更清楚地表明命名空間的作用範圍。

* 命名空間名稱應與資料夾結構一致。

  這能讓程式碼結構更清晰，並方便維護和管理專案。

## 3. 程式碼樣式

### 3.1. 關鍵字與型別

* 應使用 C# 語言關鍵字來表示內建型別 (例如，應使用 `int` 而不是 `System.Int32`)。

  使用關鍵字能使程式碼更簡潔，並符合 C# 的慣例。

* 在本地變數、參數和成員中使用預先定義的型別關鍵字。

  這能確保程式碼的一致性，並減少開發者在閱讀程式碼時的認知負擔。

* 在成員存取中使用預先定義的型別關鍵字。

  同樣是為了確保程式碼風格一致，並減少不必要的冗餘。

### 3.2. `this`

* 在程式碼中，**不**應使用 `this.` 來存取事件、欄位、方法和屬性成員，除非有明確的語法衝突。

  這能使程式碼更簡潔，並減少程式碼的噪音。

### 3.3. 括號

* 在算術二元運算子、其他二元運算子和關係二元運算子中，應**總是**使用括號以增加程式碼清晰度。

  括號能明確表達運算子的優先順序，避免程式碼出現意料之外的行為。

* 在其他運算子中，只有在**必要時**才使用括號。

  避免不必要的括號可以使程式碼更簡潔。

### 3.4. 修飾詞

* 對於非介面成員，應**總是**指定存取修飾詞 (`public`、`private`、`protected`)。

  明確指定存取修飾詞能確保程式碼的可維護性，並避免出現不必要的安全隱患。

* 建議使用以下修飾詞順序：`public`, `private`, `protected`, `internal`, `file`, `const`, `static`, `extern`, `new`, `virtual`, `abstract`, `sealed`, `override`, `readonly`, `unsafe`, `required`, `volatile`, `async`。

  統一的修飾詞順序可以使程式碼更一致，並方便閱讀。

### 3.5. 表達式層級偏好

* 應使用 `??` 空合併運算子 (Coalesce expression)。

  使用空合併運算子能使程式碼更簡潔，並避免處理空值時出現錯誤。

* 應使用集合初始器 (Collection initializer)。

  使用集合初始器能使程式碼更簡潔，並方便初始化集合。

* 應使用明確的元組名稱 (Explicit tuple names)。

  為元組明確指定名稱能使程式碼更易讀，並方便理解元組的含義。

* 應使用空值傳播運算子 (Null propagation operator, `?.`)。

  使用空值傳播運算子能簡化空值檢查，並避免出現空值錯誤。

* 應使用物件初始化器 (Object initializer)。

  使用物件初始化器能使程式碼更簡潔，並方便初始化物件。

* 當換行時，運算子應放在行首。

  這能讓程式碼更易讀，並清楚表達運算符的優先順序。

* 應優先使用自動屬性 (Auto property)。

  自動屬性能減少程式碼的冗餘，並使程式碼更簡潔。

* 當型別鬆散匹配時，應使用集合表達式 (Collection expression)。

  集合表達式能更簡潔地初始化集合，並適用於不同型別之間的轉換。

* 應使用複合賦值運算子 (例如 `+=`，`-=`)。

  複合賦值運算子能使程式碼更簡潔，並減少程式碼的冗餘。

* 在條件賦值和返回時，應使用條件表達式 (`?:`) 而非 `if/else` 語句。

  條件表達式能更簡潔地表達條件邏輯，並使程式碼更易讀。

* 在 `foreach` 迴圈中，當有明確的型別宣告時，應優先使用明確的型別轉換。

  這能確保程式碼的型別安全，並減少運行時錯誤。

* 應推斷匿名型別成員名稱和元組名稱。

  自動推斷名稱能使程式碼更簡潔，並減少程式碼的冗餘。

* 應優先使用 `is null` 檢查，而不是使用 `== null` 或 `!= null` 的方式。

  使用 `is null` 能更簡潔地檢查空值，並符合 C# 的語法慣例。

* 應使用簡化的布林表達式 (Boolean expression)。

  在格式化 .NET 程式碼時，偏好簡化的布林表達式。這意味著在可能的情況下，編輯器會建議將複雜的布林邏輯簡化。例如，將 `if (x == true)` 簡化為 `if (x)`。這樣可以使程式碼更加簡潔和易讀，同時也避免了不必要的比較運算，並提高程式碼的執行效率。

* 應使用簡化的插值字串 (Interpolated string)。

  在格式化 .NET 程式碼時，偏好簡化的插值字串。插值字串是指在字串中嵌入變數或表達式的值。例如，將 `$"{x.ToString()}"` 簡化為 `$"{x}"`。這樣可以使程式碼更加簡潔，並減少不必要的呼叫，同時也能提升程式碼的可讀性。

### 3.6. 欄位

* 如果可能，應將欄位宣告為 `readonly`。

  如果欄位的值在初始化之後不會被修改，則應宣告為 `readonly`，以確保程式碼的安全性，並防止程式碼意外修改欄位的值。

### 3.7. 參數

* 應檢查是否使用未使用的參數。

  如果參數在方法中沒有被使用，則應該移除該參數，以避免程式碼的混亂，並增加程式碼的可讀性。

### 3.8. 變數宣告

* 不應在其他地方使用 `var` 關鍵字。

   `var` 關鍵字只能用於區域變數宣告，以避免程式碼的混亂。

* 不應針對內建型別使用 `var`。

  針對內建型別 (如 `int`, `string`, `bool` 等)，應明確指定型別，以提高程式碼的可讀性。

* 在型別明顯可推斷時可以使用 `var`。

   只有在型別可以明顯從等號右邊推斷出來時，才可以使用 `var`，以避免程式碼的可讀性降低。

### 3.9. 表達式主體成員

* 應使用表達式主體來定義存取子 (accessors)。

  對於簡單的存取子 (如 `get` 和 `set`)，應使用表達式主體，以使程式碼更簡潔。

* 不應使用表達式主體來定義建構子、局部函式、方法和運算子。

  對於複雜的建構子、局部函式、方法和運算子，應使用區塊語法，以增加程式碼的可讀性。

* 應使用表達式主體來定義索引子和 lambda。

  對於簡單的索引子和 lambda，應使用表達式主體，以使程式碼更簡潔。

* 應使用表達式主體來定義屬性。

   對於簡單的屬性，應使用表達式主體，以使程式碼更簡潔。

### 3.10. 模式比對

* 應優先使用模式比對而非使用 `as` 運算子並進行空值檢查。

   模式比對能更簡潔地檢查型別和空值，並減少程式碼的冗餘。

* 應優先使用模式比對而非使用 `is` 運算子並進行型別轉換檢查。

   模式比對能更簡潔地檢查型別和進行型別轉換，並減少程式碼的冗餘。

* 應優先使用擴展屬性模式。

  擴展屬性模式能更簡潔地存取嵌套屬性，並減少程式碼的冗餘。

* 應優先使用 `not` 模式。

  `not` 模式能更簡潔地表達條件否定，並增加程式碼的可讀性。

* 應優先使用模式比對。

  在可以使用模式比對的地方，應盡量使用模式比對，以增加程式碼的可讀性。

* 應優先使用 switch 表達式。

  當需要使用 `switch` 語句時，應優先使用 `switch` 表達式，以增加程式碼的簡潔性。

### 3.11. 空值檢查

* 應使用條件委派呼叫 (`?.Invoke`)。

  條件委派呼叫能更簡潔地檢查委派是否為空，並避免出現空值錯誤。

### 3.12. 其他

* 匿名函式應宣告為 `static`。

  如果匿名函式不需要訪問外部變數，則應宣告為 `static`，以提高程式碼的性能。

* 局部函式應宣告為 `static`。

  如果局部函式不需要訪問外部變數，則應宣告為 `static`，以提高程式碼的性能。

* 結構體應宣告為 `readonly`。

  如果結構體的值在初始化之後不會被修改，則應宣告為 `readonly`，以確保程式碼的安全性，並防止程式碼意外修改結構體的值。

* 結構體成員應宣告為 `readonly`。

  如果結構體的成員的值在初始化之後不會被修改，則應宣告為 `readonly`，以確保程式碼的安全性，並防止程式碼意外修改結構體成員的值。

* 應使用大括號 (`{}`)。

  在所有控制流程語句 (如 `if`、`for`、`while` 等) 中都應使用大括號，以提高程式碼的可讀性。

* 應使用簡單的 `using` 語句。

  對於只需要釋放資源的 `using` 語句，應使用簡單的 `using` 語句，以減少程式碼的冗餘。

* 應優先使用主要建構子 (Primary Constructor)。

  如果類別或結構體只有主要的建構子，則應使用主要建構子，以減少程式碼的冗餘。

* 應優先使用最上層語句 (Top-level statements)。

   如果程式碼只包含簡單的程式碼邏輯，則應使用最上層語句，以減少程式碼的冗餘。

* 應優先使用簡單的 default 表達式。

   當需要使用 `default` 值時，應使用簡單的 `default` 表達式，以增加程式碼的可讀性。

* 應使用解構的變數宣告。

  在需要使用元組或其他可解構型別時，應使用解構的變數宣告，以增加程式碼的可讀性。

* 當型別明顯時，應使用隱式物件建立 (Implicit object creation)。

  當型別可以明顯從等號右邊推斷出來時，應使用隱式物件建立，以增加程式碼的簡潔性。

* 應使用內聯變數宣告 (Inlined variable declaration)。

   如果變數只在一個地方被使用，則應使用內聯變數宣告，以減少程式碼的冗餘。

* 應優先使用索引運算子。

  當需要使用索引存取集合時，應優先使用索引運算子，以增加程式碼的可讀性。

* 應優先使用局部函式而非匿名函式。

  如果需要使用函式，且函式不需要訪問外部變數，則應優先使用局部函式，以提高程式碼的性能。

* 應優先使用空值檢查而非型別檢查。

  在需要檢查物件是否為空時，應優先使用空值檢查，以避免出現不必要的型別錯誤。

* 應優先使用範圍運算子。

  當需要使用範圍時，應優先使用範圍運算子，以增加程式碼的可讀性。

* 應優先使用元組交換。

  當需要交換兩個變數的值時，應優先使用元組交換，以增加程式碼的簡潔性。

* 應優先使用 UTF-8 字串文字。

  當需要使用 UTF-8 字串文字時，應使用 UTF-8 字串文字，以確保程式碼的字元編碼正確。

* 應使用 `throw` 表達式。

  當需要 `throw` 例外時，應使用 `throw` 表達式，以增加程式碼的簡潔性。

* 應使用棄置字元 (discard, `_`) 變數來忽略未使用的賦值。

  當程式碼中有未使用的變數賦值時，應使用棄置字元 (`_`) 忽略，以避免程式碼的警告。

* 應使用棄置字元 (`_`) 來忽略表達式語句中未使用的值。

   當程式碼中有表達式語句，但其值不需要使用時，應使用棄置字元 (`_`) 忽略，以避免程式碼的警告。

## 4. 格式化規則

### 4.1. 換行

* `catch` 前應有換行。

  在 `try...catch` 語句中，`catch` 關鍵字之前應有一個新行，以增加程式碼的可讀性。

* `else` 前應有換行。

  在 `if...else` 語句中，`else` 關鍵字之前應有一個新行，以增加程式碼的可讀性。

* `finally` 前應有換行。

  在 `try...finally` 語句中，`finally` 關鍵字之前應有一個新行，以增加程式碼的可讀性。

* 匿名型別中的成員前應有換行。

  在匿名型別中，成員之間應有新行，以增加程式碼的可讀性。

* 物件初始化器中的成員前應有換行。

  在物件初始化器中，成員之間應有新行，以增加程式碼的可讀性。

* 在左大括號 `{` 之前，應有換行。

  在所有程式碼區塊的左大括號之前，應有一個新行，以增加程式碼的可讀性。

* 查詢表達式子句之間應有換行。

  在查詢表達式中，各個子句之間應有新行，以增加程式碼的可讀性。

### 4.2. 縮排

* 區塊內容應縮排。

  所有程式碼區塊的內容都應該向右縮排，以增加程式碼的可讀性。

* 大括號 `{}` 不應縮排。

  程式碼區塊的左大括號和右大括號，應與對應的程式碼關鍵字對齊，而不是縮排，以增加程式碼的可讀性。

* `case` 內容應縮排。

  在 `switch` 語句中，`case` 關鍵字之後的內容應向右縮排，以增加程式碼的可讀性。

* 當 `case` 內容使用區塊 `{}` 時，應縮排。

  當 `case` 關鍵字之後的內容使用大括號包圍時，應將內容向右縮排，以增加程式碼的可讀性。

* `label` 應比目前縮排少一層。

  `label` 標籤應該比目前的程式碼區塊縮排少一層，以增加程式碼的可讀性。

* `switch` 標籤應縮排。

  在 `switch` 語句中，`case` 和 `default` 標籤都應該縮排，以增加程式碼的可讀性。

### 4.3. 空格

* 轉換 (`cast`) 後不應有空格。

  型別轉換運算子和變數之間不應有空格。

* 繼承子句中的冒號 `:` 後應有空格。

  在類別繼承或介面實作中，冒號之後應有一個空格。

* 逗號 `,` 後應有空格。

  在變數宣告、方法參數和集合初始化器中，逗號之後應有一個空格。

* 點號 `.` 前後不應有空格。

  在物件成員存取時，點號前後都不應有空格。

* 控制流程語句中的關鍵字後應有空格。

  在 `if`、`for`、`while` 等控制流程語句中，關鍵字之後應有一個空格。

* `for` 語句中的分號 `;` 後應有空格。

  在 `for` 迴圈中，分號之後應有一個空格。

* 二元運算子前後應有空格。

  在二元運算子 (如 `+`、`-`、`*`、`/` 等) 前後都應有一個空格。

* 宣告語句周圍不應有空格。

  在變數宣告語句前後不應有額外空格。

* 繼承子句中的冒號 `:` 前應有空格。

  在類別繼承或介面實作中，冒號之前應有一個空格。

* 逗號 `,` 前不應有空格。

  在變數宣告、方法參數和集合初始化器中，逗號之前不應有空格。

* 開啟方括號 `[` 前不應有空格。

  在集合或索引器存取中，開啟方括號之前不應有空格。

* `for` 語句中的分號 `;` 前不應有空格。

  在 `for` 迴圈中，分號之前不應有空格。

* 空方括號 `[]` 之間不應有空格。

  在宣告空陣列或空索引時，空方括號之間不應有空格。

* 方法呼叫的空參數列表括號之間不應有空格。

  在方法呼叫時，如果參數列表為空，則括號之間不應有空格。

* 方法呼叫的名稱與左括號之間不應有空格。

  在方法呼叫時，方法名稱和左括號之間不應有空格。

* 方法呼叫的參數列表括號之間不應有空格。

  在方法呼叫時，如果參數列表不為空，則參數列表的括號之間不應有空格。

* 方法宣告的空參數列表括號之間不應有空格。

  在方法宣告時，如果參數列表為空，則括號之間不應有空格。

* 方法宣告的名稱與左括號之間不應有空格。

  在方法宣告時，方法名稱和左括號之間不應有空格。

* 方法宣告的參數列表括號之間不應有空格。

  在方法宣告時，如果參數列表不為空，則參數列表的括號之間不應有空格。

* 括號之間不應有空格。

  在括號內不應有額外空格。

* 方括號之間不應有空格。

  在方括號內不應有額外空格。

### 4.4. 換行

* 應保留單行程式碼區塊。

  在格式化 C# 程式碼時，應該保留單行的區塊。區塊通常是指用大括號 `{}` 包圍的程式碼片段，例如 `if` 語句或方法定義。如果這個區塊的內容可以放在同一行，編輯器將會保留這種單行格式，而不會自動將其展開成多行。例如，`if (x) { return true; }` 會被保留在一行內。

* 應保留單行程式碼語句。

  在格式化 C# 程式碼時，應該保留單行的語句。這意味著如果某個語句可以放在一行內，編輯器將會保留這種單行格式，而不會自動將其拆分成多行。例如，`int x = 10;` 會被保留在一行內。

## 5. 命名規則

### 5.1. 命名慣例

* **型別和命名空間** (Type and Namespace) 應使用 **PascalCase** (例如 `MyClass`, `MyNamespace`)。
* **介面** (Interface) 應使用 **IPascalCase** (例如 `IMyInterface`)。
* **型別參數** (Type Parameter) 應使用 **TPascalCase** (例如 `TMyType`)。
* **方法** (Method) 應使用 **PascalCase** (例如 `MyMethod()`)。
* **屬性** (Property) 應使用 **PascalCase** (例如 `MyProperty`)。
* **事件** (Event) 應使用 **PascalCase** (例如 `MyEvent`)。
* **本地變數** (Local Variable) 應使用 **camelCase** (例如 `myVariable`)。
* **本地常數** (Local Constant) 應使用 **camelCase** (例如 `myConstant`)。
* **參數** (Parameter) 應使用 **camelCase** (例如 `myParameter`)。
* **公開欄位** (Public Field) 應使用 **PascalCase** (例如 `MyField`)。
* **私有欄位** (Private Field) 應使用 **\_camelCase** (例如 `_myField`)。
* **私有靜態欄位** (Private Static Field) 應使用 **s\_camelCase** (例如 `s_myStaticField`)。
* **公開常數欄位** (Public Constant Field) 應使用 **PascalCase** (例如 `MyConstant`)。
* **私有常數欄位** (Private Constant Field) 應使用 **PascalCase** (例如 `MyPrivateConstant`)。
* **公開靜態 `readonly` 欄位** (Public Static `readonly` Field) 應使用 **PascalCase** (例如 `MyStaticReadonlyField`)。
* **私有靜態 `readonly` 欄位** (Private Static `readonly` Field) 應使用 **PascalCase** (例如 `MyPrivateStaticReadonlyField`)。
* **列舉** (Enum) 應使用 **PascalCase** (例如 `MyEnum`)。
* **局部函式** (Local Function) 應使用 **PascalCase** (例如 `MyLocalFunction()`)。
* **非欄位成員** (Non-Field Member) 應使用 **PascalCase** (例如 `MyMember`)。

### 5.2. 命名規則定義

此部分定義了不同符號型別的命名規則。

* **介面** (Interface)：適用於介面型別，包含 `public`、`internal`、`private`、`protected`、`protected_internal` 和 `private_protected` 的可存取性，不需要修飾詞。
* **列舉** (Enum)：適用於列舉型別，包含 `public`、`internal`、`private`、`protected`、`protected_internal` 和 `private_protected` 的可存取性，不需要修飾詞。
* **事件** (Event)：適用於事件成員，包含 `public`、`internal`、`private`、`protected`、`protected_internal` 和 `private_protected` 的可存取性，不需要修飾詞。
* **方法** (Method)：適用於方法成員，包含 `public`、`internal`、`private`、`protected`、`protected_internal` 和 `private_protected` 的可存取性，不需要修飾詞。
* **屬性** (Property)：適用於屬性成員，包含 `public`、`internal`、`private`、`protected`、`protected_internal` 和 `private_protected` 的可存取性，不需要修飾詞。
* **公開欄位** (Public Field)：適用於 `public` 和 `internal` 的欄位成員，不需要修飾詞。
* **私有欄位** (Private Field)：適用於 `private`、`protected`、`protected_internal` 和 `private_protected` 的欄位成員，不需要修飾詞。
* **私有靜態欄位** (Private Static Field)：適用於 `private`、`protected`、`protected_internal` 和 `private_protected` 的靜態欄位成員，需要 `static` 修飾詞。
* **型別和命名空間** (Type and Namespace)：適用於命名空間、類別、結構、介面和列舉，包含 `public`、`internal`、`private`、`protected`、`protected_internal` 和 `private_protected` 的可存取性，不需要修飾詞。
* **非欄位成員** (Non-Field Member)：適用於屬性、事件和方法成員，包含 `public`、`internal`、`private`、`protected`、`protected_internal` 和 `private_protected` 的可存取性，不需要修飾詞。
* **型別參數** (Type Parameter)：適用於命名空間，適用於任何可存取性，不需要修飾詞。
* **私有常數欄位** (Private Constant Field)：適用於 `private`、`protected`、`protected_internal` 和 `private_protected` 的常數欄位成員，需要 `const` 修飾詞。
* **本地變數** (Local Variable)：適用於本地變數，適用於任何可存取性，不需要修飾詞。
* **本地常數** (Local Constant)：適用於本地常數，適用於任何可存取性，需要 `const` 修飾詞。
* **參數** (Parameter)：適用於參數，適用於任何可存取性，不需要修飾詞。
* **公開常數欄位** (Public Constant Field)：適用於 `public` 和 `internal` 的常數欄位成員，需要 `const` 修飾詞。
* **公開靜態 `readonly` 欄位** (Public Static `readonly` Field)：適用於 `public` 和 `internal` 的靜態 `readonly` 欄位成員，需要 `static` 和 `readonly` 修飾詞。
* **私有靜態 `readonly` 欄位** (Private Static `readonly` Field)：適用於 `private`、`protected`、`protected_internal` 和 `private_protected` 的靜態 `readonly` 欄位成員，需要 `static` 和 `readonly` 修飾詞。
* **局部函式** (Local Function)：適用於局部函式，適用於任何可存取性，不需要修飾詞。

### 5.3. 命名風格

* **PascalCase**：首字母大寫，單字之間無分隔符號，例如 `MyClassName`。
* **IPascalCase**：首字母為 `I` 的 PascalCase，例如 `IMyInterface`。
* **TPascalCase**：首字母為 `T` 的 PascalCase，例如 `TMyType`。
* **\_camelCase**：首字母小寫，單字之間無分隔符號，並以底線 `_` 開頭，例如 `_myVariable`。
* **camelCase**：首字母小寫，單字之間無分隔符號，例如 `myVariable`。
* **s\_camelCase**：首字母為 `s_` 的 camelCase，例如 `s_myStaticVariable`。


## 6. 版本控制指南

### 6.1. Git 分支管理

* **主要分支**
  * `main`：主要發布分支，永遠保持可部署狀態
  * `develop`：開發分支，整合所有功能開發
  * `feature/*`：功能開發分支
  * `hotfix/*`：緊急修復分支
  * `release/*`：版本發布準備分支

* **分支命名規則**
  * 功能分支：`feature/[任務編號]-[簡短描述]`
    * 例：`feature/PROJ-123-user-authentication`
  * 修復分支：`hotfix/[任務編號]-[簡短描述]`
    * 例：`hotfix/PROJ-456-fix-login-crash`
  * 發布分支：`release/[版本號]`
    * 例：`release/v1.2.0`

### 6.2. Commit 訊息規範

* **格式**：
  ```
  <type>(<scope>): <subject>

  <body>

  <footer>
  ```

* **Type 類型**
  * `feat`：新功能
  * `fix`：錯誤修復
  * `docs`：文件更新
  * `style`：程式碼格式調整
  * `refactor`：重構程式碼
  * `test`：新增或修改測試
  * `chore`：建置程序或輔助工具的變動

* **範例**：
  ```
    feat(驗證): 實作 JWT 身分驗證機制

    - 新增 JWT 權杖產生功能
    - 新增權杖驗證中介軟體
    - 更新使用者登入流程

    解決問題: PROJ-123
  ```

### 6.3. 合併請求 (Pull Request) 規範

* **標題格式**：`[任務編號] 簡短描述`
* **描述內容**：
  * 功能說明或修復內容
  * 測試方式
  * 相關文件連結
  * 截圖（如適用）

## 7. 文件化規範

### 7.1. 程式碼註解

* **單行註解**：使用 `//` 進行單行註解
  ```csharp
  // 這是一個單行註解
  var user = GetCurrentUser();
  ```

* **多行註解**：使用 `/* */` 進行多行註解
  ```csharp
  /* 這是一個
     多行註解的
     示例 */
  ```

* **TODO 註解**：使用 `// TODO:` 標記待辦事項
  ```csharp
  // TODO: 實作使用者驗證機制
  ```

### 7.2. XML 文件註解

* **類別註解**：
  ```csharp
  /// <summary>
  /// 使用者服務類別，處理所有使用者相關操作
  /// </summary>
  /// <remarks>
  /// 此類別實作 IUserService 介面，提供使用者的 CRUD 操作
  /// </remarks>
  public class UserService : IUserService
  {
  ```

* **方法註解**：
  ```csharp
  /// <summary>
  /// 根據使用者 ID 取得使用者資訊
  /// </summary>
  /// <param name="userId">使用者唯一識別碼</param>
  /// <returns>使用者資訊物件，如果找不到則返回 null</returns>
  /// <exception cref="ArgumentException">當 userId 為空或格式錯誤時拋出</exception>
  public async Task<User> GetUserByIdAsync(string userId)
  {
  ```

* **屬性註解**：
  ```csharp
  /// <summary>
  /// 取得或設定使用者名稱
  /// </summary>
  /// <value>使用者的顯示名稱</value>
  public string Username { get; set; }
  ```

### 7.3. 文件化原則

* 所有公開的類別、介面、方法和屬性都必須有 XML 文件註解
* 註解應該說明「做什麼」而不是「怎麼做」
* 避免使用過於明顯的註解
* 適當使用程式碼自註解 (Self-Documenting Code)

## 8. 錯誤處理準則

### 8.1. 例外處理原則

* **使用具體的例外類別**：
  ```csharp
  // 好的做法
  throw new ArgumentNullException(nameof(user), "User cannot be null");
  
  // 避免
  throw new Exception("Invalid user");
  ```

* **自定義例外類別命名**：
  * 以 `Exception` 結尾
  * 繼承自最相關的例外基底類別
  ```csharp
  public class UserNotFoundException : Exception
  {
      public UserNotFoundException(string message) : base(message) { }
  }
  ```

### 8.2. 例外處理模式

* **使用 try-catch-finally 區塊**：
  ```csharp
  public async Task ProcessUserAsync(User user)
  {
      try
      {
          await ValidateUserAsync(user);
          await SaveUserAsync(user);
      }
      catch (ValidationException ex)
      {
          _logger.LogError(ex, "User validation failed");
          throw;
      }
      catch (DbException ex)
      {
          _logger.LogError(ex, "Database operation failed");
          throw new UserProcessingException("Failed to process user", ex);
      }
      finally
      {
          await CleanupResourcesAsync();
      }
  }
  ```

### 8.3. 日誌記錄準則

* **日誌等級使用規範**：
  * `Trace`：詳細的程式流程追蹤
  * `Debug`：開發時的偵錯資訊
  * `Information`：一般操作資訊
  * `Warning`：潛在問題警告
  * `Error`：處理失敗但應用程式可以繼續運行
  * `Critical`：需要立即處理的嚴重問題

* **日誌內容格式**：
  ```csharp
  _logger.LogError(
      ex,
      "Failed to process user {UserId}. Error: {ErrorMessage}",
      user.Id,
      ex.Message
  );
  ```

## 9. 其他

* 本規範應適用於所有 C# 程式碼。

  所有團隊成員都應遵循此規範來編寫 C# 程式碼。

* 如有任何不清楚的地方，請諮詢團隊成員。

  當您對規範有任何疑問或不確定之處時，請及時諮詢團隊成員，以確保對規範的理解一致。

* 本規範允許根據團隊需要進行更新和修改。

  本規範不是一成不變的，可以根據團隊的需要進行調整和完善。

* 版本歷程：
  * v2.0 (2025/01/05)：新增版本控制指南、文件化規範和錯誤處理準則
  * v1.0 (2024/01/01)：初始版本
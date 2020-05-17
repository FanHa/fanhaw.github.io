### 声明函数方式
```js
//  普通函数声明
function funcA (paramA, paramB) {
    // ...
}
```

### 解析function入口
```cpp
// src/parsing/parser-base.h
template <typename Impl>
typename ParserBase<Impl>::StatementT
ParserBase<Impl>::ParseStatementListItem() {
  switch (peek()) {
    case Token::FUNCTION:
      // 解析语句遇到‘function’ Token时,进入HoistableSeclaration逻辑;
      // Hoistable表明function声明是变量提升,即function在同一个上下文中声明可以落后于使用
      return ParseHoistableDeclaration(nullptr, false);
    // ...
  }
}

// 封装ParseHoistableDeclaration,消费掉‘function’Token,初始化一些变量并调用重载版的ParseHoistableDeclaration
template <typename Impl>
typename ParserBase<Impl>::StatementT
ParserBase<Impl>::ParseHoistableDeclaration(
    ZonePtrList<const AstRawString>* names, bool default_export) {
  Consume(Token::FUNCTION);

  int pos = position();
  ParseFunctionFlags flags = ParseFunctionFlag::kIsNormal;
  if (Check(Token::MUL)) {
    flags |= ParseFunctionFlag::kIsGenerator;
  }
  return ParseHoistableDeclaration(pos, flags, names, default_export);
}


template <typename Impl>
typename ParserBase<Impl>::StatementT
ParserBase<Impl>::ParseHoistableDeclaration(
    int pos, ParseFunctionFlags flags, ZonePtrList<const AstRawString>* names,
    bool default_export) {
  // ...

  IdentifierT name;
  FunctionNameValidity name_validity;
  IdentifierT variable_name;
  if (peek() == Token::LPAREN) {
    // ‘function’Token后面直接接‘(‘的情形只有在export时才有效
    if (default_export) {
      impl()->GetDefaultStrings(&name, &variable_name);
      name_validity = kSkipFunctionNameCheck;
    } else {
      ReportMessage(MessageTemplate::kMissingFunctionName);
      return impl()->NullStatement();
    }
  } else {
    // 一般形式的 function aaa(){}

    // 判断函数名是否是系统保留名称
    bool is_strict_reserved = Token::IsStrictReservedWord(peek());
    // 获取函数名
    name = ParseIdentifier();
    name_validity = is_strict_reserved ? kFunctionNameIsStrictReserved
                                       : kFunctionNameValidityUnknown;
    variable_name = name;
  }

  FuncNameInferrerState fni_state(&fni_);
  impl()->PushEnclosingName(name);

  // 获取函数类型, 因为*generator函数, async函数, function里的方法(method)都是走这一解析流程;
  // 需要在这里做一个标记便于后面走不同的解析流程
  FunctionKind function_kind = FunctionKindFor(flags);

  // 这里就是对function内容的解析;
  // 这个ParseFunctionLiteral解析是一层封装,v8对函数的解析分为 parse(贪婪) 和 preparse(保守);
  // parse解析的更彻底;
  // preparse解析得相对肤浅,只解析必要信息,只有在后面真正使用这个函数时才会激活真正的parse,类似于“懒加载”
  FunctionLiteralT function = impl()->ParseFunctionLiteral(
      name, scanner()->location(), name_validity, function_kind, pos,
      FunctionSyntaxKind::kDeclaration, language_mode(), nullptr);

  // ...
  // 将函数名与函数内容绑定作具体声明
  return impl()->DeclareFunction(variable_name, function, mode, kind, pos,
                                 end_position(), names);
}
```

#### ParseFunctionLiteral 的贪心与延迟
```cpp
// src/parsing/parser.cc
FunctionLiteral* Parser::ParseFunctionLiteral(
    const AstRawString* function_name, Scanner::Location function_name_location,
    FunctionNameValidity function_name_validity, FunctionKind kind,
    int function_token_pos, FunctionSyntaxKind function_syntax_kind,
    LanguageMode language_mode,
    ZonePtrList<const AstRawString>* arguments_for_wrapped_function) {
  // ...
  int pos = function_token_pos == kNoSourcePosition ? peek_position()
  // ...
  // 根据上下文设置贪婪hint,后面需要用这个hint来决定是否贪婪还是延迟解析
  FunctionLiteral::EagerCompileHint eager_compile_hint =
      function_state_->next_function_is_likely_called() || is_wrapped
          ? FunctionLiteral::kShouldEagerCompile
          : default_eager_compile_hint();

  // 贪婪还是延迟解析需要一系列上下文共同决定..
  const bool is_lazy =
      eager_compile_hint == FunctionLiteral::kShouldLazyCompile;
  const bool is_top_level = AllowsLazyParsingWithoutUnresolvedVariables();
  const bool is_eager_top_level_function = !is_lazy && is_top_level;
  const bool is_lazy_top_level_function = is_lazy && is_top_level;
  const bool is_lazy_inner_function = is_lazy && !is_top_level;

  // 根据前面的hint设置当前解析是否是需要延迟解析的inner函数;
  // 只有在开启了parseLazily 并且前面的 ‘is_lazy_inner_function’ 设置为true时,将‘should_preparse_inner’ 设置为true
  const bool should_preparse_inner = parse_lazily() && is_lazy_inner_function;

  // 现代js解析器已经超越了传统的“js是单线程运行”的禁锢观念,比如这里可以先只简单的preparse函数,然后把详细的解析任务分发给一个平行的线程
  bool should_post_parallel_task =
      parse_lazily() && is_eager_top_level_function &&
      FLAG_parallel_compile_tasks && info()->parallel_tasks() &&
      scanner()->stream()->can_be_cloned_for_parallel_access();

  // 根据前面的解析结果设置‘should_preparse‘值
  bool should_preparse = (parse_lazily() && is_lazy_top_level_function) ||
                         should_preparse_inner || should_post_parallel_task;

  // 初始化函数body里的基本信息;
  // 将function的body里的内容初始化为ScopedPtrList
  ScopedPtrList<Statement> body(pointer_buffer());
  int expected_property_count = 0;
  int suspend_count = -1;
  int num_parameters = -1;
  int function_length = -1;
  bool has_duplicate_parameters = false;
  int function_literal_id = GetNextFunctionLiteralId();
  ProducedPreparseData* produced_preparse_data = nullptr;

  // This Scope lives in the main zone. We'll migrate data into that zone later.
  Zone* parse_zone = should_preparse ? &preparser_zone_ : zone();
  // 为函数创建一个新的Scope, GC相关??
  DeclarationScope* scope = NewFunctionScope(kind, parse_zone);
  SetLanguageMode(scope, language_mode);

  scope->set_start_position(position());

  // 如果上面的分析决定应该使用延迟(preparse),则调用SkipFunction;
  // 但是这个SkipFunction依然需要参考当前的各个中泰条件,是不确定能否延迟解析成功,仍然会有返回false的情况
  bool did_preparse_successfully =
      should_preparse &&
      SkipFunction(function_name, kind, function_syntax_kind, scope,
                   &num_parameters, &function_length, &produced_preparse_data);

  if (!did_preparse_successfully) {
    // 实在不能延迟了,消费掉‘(’,开始动手解析(ParseFunction)
    if (should_preparse) Consume(Token::LPAREN);
    should_post_parallel_task = false;
    ParseFunction(&body, function_name, pos, kind, function_syntax_kind, scope,
                  &num_parameters, &function_length, &has_duplicate_parameters,
                  &expected_property_count, &suspend_count,
                  arguments_for_wrapped_function);
  }

  // ...

  // 以前面的解析结果初始化一个需要返回的FunctionLiteral 数据结构
  FunctionLiteral* function_literal = factory()->NewFunctionLiteral(
      function_name, scope, body, expected_property_count, num_parameters,
      function_length, duplicate_parameters, function_syntax_kind,
      eager_compile_hint, pos, true, function_literal_id,
      produced_preparse_data);
  function_literal->set_function_token_position(function_token_pos);
  function_literal->set_suspend_count(suspend_count);

  // ...
  if (should_post_parallel_task) {
    // 当前面采用平行解析(即当前只preparse,然后把具体parse分配给另一个线程去作)方案时,把任务入队
    info()->parallel_tasks()->Enqueue(info(), function_name, function_literal);
  }

  return function_literal;
}

```

#### 延迟(SkipFunction)

#### 贪心(ParseFunction)
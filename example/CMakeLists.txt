add_python_unit_test("add_eq" "example/add_eq.py" "python")
if(BUILD_WITH_CPPAD_CODEGEN_BINDINGS)
  add_python_unit_test("cppadcg_c_codegen" "example/cppadcg_c_codegen.py"
                       "python")
endif(BUILD_WITH_CPPAD_CODEGEN_BINDINGS)

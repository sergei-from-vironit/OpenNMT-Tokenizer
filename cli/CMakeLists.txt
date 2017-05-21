hunter_add_package(Boost COMPONENTS program_options)
find_package(Boost COMPONENTS program_options)

add_executable(tokenize
  tokenize.cc
  )
target_link_libraries(tokenize
  ${PROJECT_NAME}
  Boost::program_options
  )

add_executable(detokenize
  detokenize.cc
  )
target_link_libraries(detokenize
  ${PROJECT_NAME}
  Boost::program_options
  )

install(
  TARGETS tokenize detokenize
  DESTINATION bin/
  )

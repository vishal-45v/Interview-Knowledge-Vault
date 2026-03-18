# Chapter 09 — Testing Rails: Theory Questions

---

**Q1.** What are the different types of tests in a Rails app? Explain the testing pyramid and how it applies to Rails. What is the recommended distribution of unit, integration, and system tests?

**Q2.** What is RSpec? How does it differ from Minitest? What are the key RSpec DSL constructs: `describe`, `context`, `it`, `let`, `let!`, `before`, `subject`?

**Q3.** What is FactoryBot? How do factories differ from fixtures? What are traits, sequences, and associations in FactoryBot?

**Q4.** Explain the difference between `build`, `build_stubbed`, `create`, and `create_list` in FactoryBot. What are the performance implications?

**Q5.** What is Capybara? What is the difference between `have_text`, `have_content`, `have_css`, and `have_selector`? When do you use `find` vs `all`?

**Q6.** What is a request spec in RSpec? How does it differ from a controller spec? Why are request specs preferred in Rails 5+?

**Q7.** What is a system test in Rails? How does it differ from a feature spec? What driver do you use for JavaScript-heavy tests?

**Q8.** What is `let` lazy evaluation in RSpec? What is the difference between `let` and `let!`? What is the n+1 gotcha with `let` in nested examples?

**Q9.** How do you test background jobs in Rails? What is `perform_enqueued_jobs` and `assert_enqueued_with`? How does `perform_later` behave in test mode?

**Q10.** What are shared examples and shared contexts in RSpec? When is it appropriate to use them vs duplicating test code?

**Q11.** How do you test mailers in Rails? What is `ActionMailer::Base.deliveries`? How do you use `have_enqueued_mail`?

**Q12.** What is VCR (cassettes)? When would you use it vs WebMock? What are the tradeoffs of each approach for testing external HTTP calls?

**Q13.** Explain the `database_cleaner` gem. What are the different strategies: `transaction`, `truncation`, `deletion`? When would you use each?

**Q14.** What is test isolation? Why is it important? What are the common sources of test pollution in Rails test suites?

**Q15.** How do you test Pundit policies? What helpers does Pundit provide for policy specs?

**Q16.** What is `travel_to` and `freeze_time`? How do you test time-dependent behavior in Rails?

**Q17.** What is a stub vs a mock vs a spy in RSpec? When should you prefer each? What are the risks of over-mocking?

**Q18.** What is code coverage? How do you use SimpleCov with Rails? What is a good coverage target and what should you be cautious about with 100% coverage as a goal?

**Q19.** How do you speed up a slow test suite? What are the most common causes of slow Rails tests?

**Q20.** What is parallel testing in Rails? How does `parallelize(workers: :number_of_processors)` work? What database setup is required?

---

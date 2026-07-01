# Contribution 1: BUG: keep_warning_stat argument in pm.sample does nothing

**Contribution Number:** 1  
**Student:** Omesh Sana
**Issue:** https://github.com/pymc-devs/pymc/issues/8321
**Status:** Phase I - Completed

---

## Why I Chose This Issue

I chose this PyMC bug because it sits at the intersection of Bayesian machine learning and practical library engineering, which fits my ML coursework and prior projects in model training and evaluation. It’s a focused, well-described behavior bug around pm.sample and the keep_warning_stat argument, so I can make a concrete, testable change instead of a vague feature.

This issue also lets me dig into how PyMC integrates different samplers (like the pymc and nutpie samplers) and how inference results are stored in InferenceData. I’m hoping to get hands-on experience with library internals, test design, and debugging subtle API behavior that users rely on when monitoring divergences and sampler warnings.

---

## Understanding the Issue

### Problem Description

In PyMC’s pm.sample function, the keep_warning_stat parameter is supposed to control whether certain warning-related statistics are stored in the returned InferenceData. For the pymc sampler, this parameter works as documented, but for the nutpie sampler, it currently has no effect: the divergence_message field is present even when keep_warning_stat=False.

### Expected Behavior

When calling pm.sample with keep_warning_stat=False, the resulting InferenceData.sample_stats should not include the divergence_message field, regardless of whether the sampler is pymc or nutpie. The behavior should be consistent across sampler backends so that users can rely on keep_warning_stat to drop these warning statistics when they don’t want to store them.

### Current Behavior

With the nutpie sampler, calling:
- pm.sample(..., keep_warning_stat=False, nuts_sampler="nutpie") still produces an InferenceData where divergence_message is present in idata.sample_stats.
- By contrast, with nuts_sampler="pymc", keep_warning_stat=False correctly removes that field, while keep_warning_stat=True keeps it.
So the keep_warning_stat argument is effectively ignored in the nutpie code path.

### Affected Components

- The implementation of pm.sample and the logic that wires keep_warning_stat into sampler-specific code.
- The integration between PyMC and the nutpie sampler, especially where InferenceData (and its sample_stats) is constructed.
- Unit tests or integration tests that cover keep_warning_stat behavior for different samplers

---

## Reproduction Process

### Environment Setup

I will set up a local development environment with:
- A recent Python version supported by PyMC.
- A clone of pymc-devs/pymc with a dedicated feature branch for this issue.
- The dependencies listed in PyMC’s contributor docs (including nutpie) installed via pip or conda as recommended.
If there are environment issues (e.g., installing nutpie or getting ArviZ/InferenceData to work), I’ll note the errors and resolutions (such as pinning versions or using the recommended environment YAML) here once I’ve gone through setup on my machine.

### Steps to Reproduce

1. Define a small PyMC model (e.g., a simple Normal model) in a Python script or notebook.
2. Call pm.sample(keep_warning_stat=False, nuts_sampler="nutpie", random_seed=0) and obtain idata_nutpie.
3. Check idata_nutpie["sample_stats"].keys() and observe that divergence_message is still present.
4. Call pm.sample(keep_warning_stat=False, nuts_sampler="pymc", random_seed=0) to obtain idata_pymc.
5. Check idata_pymc["sample_stats"].keys() and observe that divergence_message is absent as expected.

### Reproduction Evidence
https://github.com/omesh-s/pymc.git
Commit showing reproduction:
- I will create a minimal script/test in my fork under a dedicated branch (e.g., bug-keep-warning-stat-nutpie) and link the commit here once pushed.
Screenshots/logs:
- I will capture the output of the keys() check for both samplers to show the difference.
My findings:
- I expect to confirm that the nutpie path does not respect keep_warning_stat, while the pymc path does, and to identify where in the nutpie integration the flag is ignored.

---

## Solution Approach

### Analysis

The root cause appears to be that the keep_warning_stat parameter is only honored in the code path for the pymc sampler, but not in the nutpie sampler integration. Somewhere between running nutpie and converting results to InferenceData, the code always includes divergence_message in sample_stats without checking keep_warning_stat.
I’ll need to trace how pm.sample passes keep_warning_stat down to sampler-specific code and where sample statistics are assembled for nutpie.

### Proposed Solution

At a high level:
- Locate the nutpie-specific sampling code and the point where InferenceData.sample_stats is constructed.
- Mirror the logic used in the pymc sampler to respect keep_warning_stat, specifically controlling whether divergence_message (and potentially related warning stats) are included.
- Add regression tests that cover all combinations of samplers (pymc vs nutpie) and keep_warning_stat values (True vs False), verifying the presence or absence of divergence_message.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** pm.sample exposes keep_warning_stat to let users drop warning-related statistics from InferenceData.sample_stats, but the nutpie sampler path ignores this flag and always keeps divergence_message.

**Match:** 
- Examine how the pymc sampler path implements keep_warning_stat.
- Look for existing patterns where fields are conditionally included in sample_stats or other InferenceData groups based on flags.

**Plan:** 
1. Identify the function(s) that implement sampling with nuts_sampler="nutpie" and where they construct or post-process InferenceData.
2. Compare that logic to the pymc sampler path to see how keep_warning_stat is used there.
3. Add or adjust code in the nutpie path so that when keep_warning_stat=False, divergence_message (and any other warning stats that mirror this behavior) are removed from sample_stats before returning InferenceData.
4. Implement tests:
  - A small model test that runs pm.sample with both samplers and both keep_warning_stat values.
  - Assertions on idata["sample_stats"].keys() to check the presence/absence of divergence_message.
5. Run the PyMC test suite (or the relevant subset) to ensure no regressions.
6. Update any relevant documentation or docstrings if they mention keep_warning_stat behavior by sampler.


**Implement:** I’ll work on a dedicated branch in my fork (e.g., bugfix/keep-warning-stat-nutpie) and reference commits as I go.

**Review:** 
Before opening the PR, I’ll check that:
- The contribution follows the project’s coding style and contributor guidelines.
- Tests are added/updated and pass locally.
- The commit messages and PR description clearly reference issue #8321 and explain the behavior change.

**Evaluate:**
I’ll verify that the fix works by:
- Rerunning the reproduction steps and observing the corrected behavior.
- Running the test suite (or a sufficient subset) to ensure nothing else is broken.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: pm.sample(keep_warning_stat=False, nuts_sampler="pymc") does not include divergence_message in sample_stats
- [ ] Test case 2: pm.sample(keep_warning_stat=True, nuts_sampler="pymc") does include divergence_message
- [ ] Test case 3: pm.sample(keep_warning_stat=False, nuts_sampler="nutpie") does not include divergence_message after the fix
- [ ] Test case 4: pm.sample(keep_warning_stat=True, nuts_sampler="nutpie") does include divergence_message

### Integration Tests

- [ ] Integration scenario 1: Run pm.sample with a slightly more complex model under both samplers and ensure keep_warning_stat consistently controls sample_stats fields.
- [ ] Integration scenario 2: Ensure that existing functionality relying on divergence_message when keep_warning_stat=True still works for both samplers.

### Manual Testing

- Run the reproduction script before and after the fix to visually inspect sample_stats.keys().
- Possibly log or print out InferenceData summaries to confirm expected fields and that sampling still converges normally

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

----

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]

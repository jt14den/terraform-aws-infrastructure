# Infrastructure as Code for Library Services

A Carpentries Workbench lesson with a six-episode public Potree/Ansible core and
a separate Dataverse/Terraform operational case study. Core learners need no AWS
credentials or private repository access. See [Setup](learners/setup.md).

**Rendered lesson:** [www.tim-dennis.com/terraform-aws-infrastructure](https://www.tim-dennis.com/terraform-aws-infrastructure/)

## Local validation

Use the established R/Workbench environment (also described in `.devcontainer/`).
CI uses `sandpaper::validate_lesson()` and builds lesson markdown. To validate and
build locally without deployment:

```bash
Rscript -e 'sandpaper::validate_lesson(); sandpaper::build_lesson(preview = FALSE)'
```

The build writes ignored output under `site/`. Do not use the CI deployment entry
point for local checks. Building the prose does not execute Ansible labs.

## Contributing

Contributions are welcome. The full guidelines are in  
[`CONTRIBUTING.md`](CONTRIBUTING.md).

Ways to contribute:

- Fix errors or unclear explanations
- Suggest or add new episodes
- Test the lesson and report issues
- Improve learner exercises

Please open an issue before submitting large changes.

## Contact

For questions about this lesson:

- **Tim Dennis**  
  tdennis@library.ucla.edu  
  Director, Data Science Center, UCLA Library

## Credits and acknowledgments

This lesson was created using:

- The Carpentries Workbench template  
  https://github.com/carpentries/workbench-template-md
- Community documentation and practices from the Terraform and AWS ecosystems

Thank you to the Carpentries community for instructional design guidance.

## Citation

If you use or adapt this lesson, please cite it following the information in  
[`CITATION.cff`](CITATION.cff).

## License

This lesson is licensed under **CC-BY 4.0**.  
See [`LICENSE.md`](LICENSE.md) for details.

# llm-d Release Process

This document outlines the release process for llm-d to enable consistent, high-quality releases.

## Overview

llm-d releases are coordinated efforts that involve:
- Feature and bugfix selection with community input
- Component releases from individual teams
- Integration and testing across guides
- Final tagging and container image publication

**Typical Release Timeline:** 2-3 weeks from issue selection to final tag

## Release Phases

### Phase 1: Issue Selection and Planning (Week -2)

#### 1.1 Community Input Period (7 days before finalization)

**Goal:** Gather community feedback on potential features and priorities before finalizing the release scope.

**Process:**
1. Post a Slack message in the llm-d community channel announcing the upcoming release
2. Request community input on:
   - Feature requests they'd like to see
   - Pain points or bugs that need addressing
   - Use cases that need better support
3. Keep the feedback window open for 7 days
4. Document all community suggestions in a tracking issue

#### 1.2 Lead Review and Prioritization

**Participants:** Project leads

**Process:**
1. Consult community suggestions and build an internal roadmap
2. Evaluate:
   - Technical feasibility within release timeframe
   - Alignment with project goals
   - Resource availability
3. Decide on:
   - Features to include
   - Bugfixes to prioritize
   - Architecture components to add/update
   - New or updated guides

#### 1.3 Draft Release Issue

**Owner:** Release coordinator

**Process:**
1. Create a GitHub issue titled `Release vX.Y.Z`
2. Document all selected items organized by:
   - **New Features**
   - **Enhancements**
   - **Bug Fixes**
   - **Component Updates**
   - **Documentation/Guide Updates**
   - **Breaking Changes** (if any)
3. Assign owners to each item
4. Set target dates:
   - Component releases deadline: 5 days before final release
   - Integration PR deadline: 2 days before final release
   - Final tag date
5. Post the draft release issue to Slack for visibility
6. Finalize after any last feedback

Examples of previous release issues include:
- v0.5.0: https://github.com/llm-d/llm-d/issues/517
- v0.4.0: https://github.com/llm-d/llm-d/issues/347
- v0.3.0: https://github.com/llm-d/llm-d/issues/146

Ideally both minor and patch releases should have a roadmap, but it is a requirement that minor releases have one.

### Phase 2: Feature Development (Weeks -2 to -1)

**Process:**
1. Teams work on assigned features and bugfixes
2. Code review and testing for each PR
3. Merge to main branch after approval
4. Update release issue as items complete
5. Communicate blockers or delays in Slack

**Best Practices:**
- Keep PRs focused and reviewable
- Include tests with new features
- Update relevant documentation
- Tag PRs with release milestone

### Phase 3: Component Releases (3-5 days before release)

**Goal:** Individual component teams release their updates and document integration changes.

#### 3.1 Component Release Process

**Each Component Team Must:**
1. Create component release (tag, GitHub release, container images)
2. Document changes in component release notes
3. **Highlight integration changes:**
   - New/changed configuration parameters
   - Breaking changes
   - New container image tags
   - Required Kubernetes version changes
   - New dependencies
4. Create either:
   - **Slack thread** documenting changes and new image versions, OR
   - **GitHub issue** with same information
5. Notify guide stakeholders who depend on the component
6. **Remain available** to support guide integration for 5 days

**Component Teams:**
- llm-d-inference-scheduler team
- llm-d-modelservice team
- Gateway provider teams
- Other component owners


#### 3.2 Component Release Checklist

- [ ] Version bumped in all relevant files
- [ ] CHANGELOG.md updated
- [ ] Git tag created (vX.Y.Z format)
- [ ] GitHub release created with notes
- [ ] Container images built and pushed
- [ ] Helm chart updated (if applicable)
- [ ] Integration changes documented
- [ ] Guide owners notified
- [ ] Posted to Slack or GitHub issue

### Phase 4: Guide Integration (3-5 days before release)

**Goal:** Update all guides to consume new component versions and validate end-to-end functionality.

#### 4.1 Integration PR Creation

**Process:**
1. Guide owners work with component teams to:
   - Update component image references
   - Incorporate configuration changes
   - Update documentation for new features
   - Update example YAMLs and Helm values
2. **Update inference image reference:**
   - Specify what the new inference image will be once the repo is tagged
   - Format: `registry/llm-d-inference:vX.Y` (or specific component version)
3. Test guides end-to-end with new component versions
4. Document any migration steps for users
5. Create PR against main branch

**Integration PR Must Include:**
- Updated component image tags throughout guides
- Updated Helm chart values/examples
- Future inference image tag (post-release)
- Updated prerequisites (if changed)
- Migration notes (if breaking changes)
- Test results or validation notes

#### 4.2 Integration Testing

**Required Testing:**
- [ ] Deploy guide from scratch following documentation
- [ ] Verify all components start successfully
- [ ] Run basic inference tests
- [ ] Verify monitoring/observability still works
- [ ] Test any new features introduced
- [ ] Validate on at least one target accelerator type

**Document Results:**
- Accelerator tested:
- Kubernetes version:
- Any issues encountered:
- Performance notes:

### Phase 5: Final Release (Release Day)

**Owner:** Release coordinator

#### 5.1 Pre-Release Checklist

- [ ] All features complete and merged
- [ ] All component releases published
- [ ] Integration PR reviewed and merged
- [ ] Release notes drafted
- [ ] Breaking changes documented
- [ ] Migration guide ready (if needed)
- [ ] Announcement drafted

#### 5.2 Create Release Tag

**Process:**
1. Ensure main branch is in desired state
2. Create annotated git tag:
   ```bash
   git tag -a vX.Y -m "Release vX.Y"
   git push origin vX.Y
   ```
3. Verify tag triggers container image builds
4. Monitor CI/CD pipeline for:
   - Container image builds
   - Image pushes to registry
   - Any automated tests

#### 5.3 Container Image Publication

**Automated via Git Tag:**
- Inference container images built automatically when tag is pushed
- Images published to container registry with vX.Y tag
- Verify images are available and pullable

**Manual Verification:**
```bash
docker pull registry/llm-d-inference:vX.Y
docker inspect registry/llm-d-inference:vX.Y
```

#### 5.4 GitHub Release Creation

**Process:**
1. Go to GitHub Releases page
2. Create new release from tag vX.Y
3. Use release notes from drafted content
4. Organize by sections:
   - **Highlights** - 2-3 sentence summary
   - **What's New** - major features
   - **Enhancements** - improvements to existing features
   - **Bug Fixes**
   - **Component Versions** - list all component versions included
   - **Breaking Changes** - with migration guidance
   - **Known Issues** - if any
   - **Community Contributions** - highlight community input
5. Include upgrade instructions
6. Link to relevant guides and documentation
7. Publish release

#### 5.5 Release Announcement

**Channels:**
1. **Slack:** Post in main channel with highlights and link
2. **GitHub Discussions:** Create announcement post
3. **Blog:** (If major release) Coordinate with marketing
4. **Social Media:** (If major release) Coordinate with marketing
5. **Mailing List:** Send to llm-d-contributors Google Group

**Announcement Template:**
```
<� llm-d vX.Y Released!

We're excited to announce llm-d vX.Y with [key highlight].

**Highlights:**
- Feature 1: [brief description]
- Feature 2: [brief description]
- Enhancement: [brief description]

**Component Versions:**
- llm-d-inference-scheduler: vX.Y
- llm-d-modelservice: vX.Y
- [other components]

**Get Started:**
- Release Notes: [link]
- Upgrade Guide: [link]
- Documentation: [link]

**Thanks to Contributors:**
Special thanks to [community contributors] and everyone who provided feedback!

Full release notes: [link]
```

## Release Cadence

**Target:** One release every 6-8 weeks

**Schedule:**
- Week -3: Begin community input period
- Week -2: Finalize release issue, begin development
- Week -1: Component releases (by Wednesday)
- Week 0: Integration, testing, final release (Friday)

## Roles and Responsibilities

### Release Coordinator
- Owns overall release timeline
- Creates and maintains release issue
- Coordinates across teams
- Creates final tag and GitHub release
- Posts announcements

### Project Leads
- Review community input
- Prioritize features
- Make final scope decisions
- Approve release content

### Component Team Leads
- Own component releases
- Document integration changes
- Support guide integration
- Communicate with release coordinator

### Guide Owners
- Integrate component updates
- Test and validate guides
- Update documentation
- Report integration issues

### Community Members
- Provide feature suggestions
- Test beta/RC releases (if applicable)
- Report issues
- Contribute PRs

## Version Numbering

Follow Semantic Versioning (semver): `MAJOR.MINOR.PATCH`

- **MAJOR:** Breaking changes, major architectural changes
- **MINOR:** New features, significant enhancements (backwards compatible)
- **PATCH:** Bug fixes, minor improvements (backwards compatible)

**Examples:**
- v0.4.0 � v0.5.0: New feature release
- v0.5.0 � v0.5.1: Bugfix release
- v0.5.1 � v1.0.0: Major release with breaking changes

## Hotfix Process

For critical bugs requiring immediate release between regular releases:

1. Create hotfix branch from release tag
2. Fix bug with minimal changes
3. Create patch release (increment PATCH version)
4. Follow abbreviated release process:
   - Create release tag
   - Publish container images
   - Create GitHub release
   - Announce in Slack
5. Cherry-pick fix to main if needed

## Post-Release

**Within 1 Week:**
- [ ] Monitor Slack/GitHub for user issues
- [ ] Address any critical bugs found
- [ ] Collect feedback on release process
- [ ] Update release process documentation if needed
- [ ] Plan next release

**Retrospective Questions:**
- What went well?
- What could be improved?
- Were timelines realistic?
- How was communication?
- What blockers did we hit?

## Communication Channels

- **Slack:** Daily updates, quick questions, announcements
- **GitHub Issues:** Feature tracking, bug reports, release issue
- **GitHub Discussions:** Release planning, architectural decisions
- **Google Group:** Architecture docs, calendar invites
- **Weekly Standup:** Wednesday 12:30 PM ET - release status updates

## Tools and Automation

**Current:**
- GitHub Actions for CI/CD
- Container registry for image hosting
- Helm for packaging
- Git tags trigger image builds

**Future Improvements:**
- Automated release notes generation
- Automated component version tracking
- Release readiness dashboard
- Automated upgrade testing

## Related Documentation

- [Contributing Guidelines](CONTRIBUTING.md)
- [Project Overview](PROJECT.md)
- [Roadmap](https://github.com/llm-d/llm-d/issues/146)
- [Guides](./guides/README.md)

---

**Questions or suggestions for this process?** Open an issue or discuss in Slack #llm-d-contributors

# Architecture Notes

## Overview

The core construct is a CDK TypeScript stack (TapStack) that provisions a VPC with two subnets per environment, selecting non-overlapping CIDR ranges based on an environment suffix passed at synthesis time. [from-code] The CIDR selection logic lives in a static method inside TapStack, making the mapping auditable and testable without touching AWS. [from-code] What is non-obvious here is the decision to encode network topology as pure TypeScript logic rather than CDK context variables or SSM parameters, which eliminates an entire class of cross-environment lookup failures at the cost of making CIDR changes a code change. [inferred]

## Key Decisions

- Hardcoding CIDRs in TypeScript means any network expansion (e.g. adding a /16 for a DR region) requires a code change, PR review, and redeployment — but it also means the CIDR map is version-controlled and diffable, unlike SSM-stored values that can drift silently. [from-code]
- Two subnets per environment is a minimal footprint that avoids NAT Gateway costs in dev/staging but gives you no AZ redundancy on those tiers — a subnet failure in dev will kill CI pipelines. [inferred]
- Using a single CDK stack for all environment variants keeps the blast radius of a misconfigured synth to one account/region, but it also means you cannot isolate prod deployment permissions from staging at the CDK app level without additional stack splitting. [inferred]
- Static CIDR method in TapStack is unit-testable (npm run test:unit), which is a real operational win — most teams only discover CIDR conflicts after a VPC peer fails in prod, not in a test run. [from-code]
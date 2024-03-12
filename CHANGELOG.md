# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## 2024-03-12

### Fixed

- Changed Network Policy for communication with LDAP and DB and make it non-blocking for operators

## 2024-03-04

### Added

- Command to be able to see hidden attributed in OpenLDAP
- Proper Network Policies which would be needed in real deployments

### Changed

- Updated to CP4BA 23.0.2 iFix 2
- Changed OpenLDAP to Bitnami edition which allows for rootless deployment
- Changed PostgreSQL to Bitnami edition which allows for rootless deployment

### Fixed

- Wrong project name for Test environment in readiness verification stage
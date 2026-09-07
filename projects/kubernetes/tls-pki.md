# TLS and Home Lab PKI

## Overview

Internal Kubernetes services under `*.home.lab` use a wildcard TLS certificate served by Traefik.

## Certificate

Wildcard:

`*.home.lab`

Certificate chain:

Home Lab PKI Root CA
↓
Home Lab PKI Intermediate CA
↓
*.home.lab

The currently available certificate bundle contains:

- `*.home.lab`
- Home Lab PKI Intermediate CA

The Root CA certificate is not currently retained as a separate accessible certificate.

## Kubernetes

The wildcard certificate is stored as a Kubernetes TLS Secret:

`traefik/wildcard-home-lab`

Traefik uses this Secret as its default TLS certificate through the default `TLSStore`.

This allows services such as:

- `vaultwarden.home.lab`
- `rancher.home.lab`

to use the wildcard certificate without requiring an individual TLS Secret for every service.

## Client Trust

For now, clients may directly trust the Home Lab PKI Intermediate CA.

A future PKI project should create/recover a proper long-lived Root CA trust anchor and establish an automated method for distributing the public Root CA certificate to managed devices.

## Security

Private keys must not be committed to Git.

Public CA certificates may be stored with configuration or documentation because they are not secret.
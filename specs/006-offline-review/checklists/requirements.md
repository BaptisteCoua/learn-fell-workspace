# Specification Quality Checklist: Révision hors ligne

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-10-01
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Notes

- Choix du développeur (prise de besoin) : cartes dues aujourd'hui et dans les 7 jours ; texte seulement, sans images ; une réponse compte à sa date ; la première réponse pour une échéance l'emporte.
- Spec validée par le développeur le 2026-10-01, FR-013 gardé tel quel (recalcul de la progression quand une réponse hors ligne plus ancienne arrive après une réponse en ligne), plutôt que d'ignorer la réponse en retard.
- Valeurs par défaut relues : FR-013 annule une réponse en ligne plus récente quand une réponse hors ligne plus ancienne arrive (recalcul de la progression) ; FR-015 borne la date de l'appareil ; FR-018 avertit avant une déconnexion qui perdrait des réponses ; les données hors ligne sont effacées à la déconnexion.

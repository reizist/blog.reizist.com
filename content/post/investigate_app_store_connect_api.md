---
title: "Investigate_app_store_connect_api"
date: 2021-03-29T16:37:47+09:00
draft: true
---

fastlane/spaceshipで使っていたAPIの代替を目的としてApp Store Connect APIについて
`app_store_connect` gem を使って調べたメモ。

<!-- more -->



## endpoint一覧

```
client = AppStoreConnect::Client.new
client.web_service_endpoint_aliases
[:delete_user,
 :delete_visible_app,
 :delete_bundle_id_capability,
 :delete_user_invitation,
 :delete_certificate,
 :delete_bundle_id,
 :delete_profile,
 :delete_beta_tester,
 :delete_beta_tester_beta,
 :delete_beta_tester_builds,
 :delete_beta_tester_apps,
 :delete_beta_group,
 :delete_beta_group_beta_testers,
 :delete_beta_group_builds,
 :delete_app_beta_testers,
 :delete_beta_app_localization,
 :delete_build_beta_group,
 :delete_build_individual_testers,
 :delete_beta_build_localizations,
 :create_certificate,
 :create_profile,
 :create_device,
 :create_user_invitation,
 :users,
 :user,
 :user_visible_apps,
 :user_relationships_visible_apps,
 :user_invitation,
 :user_invitations,
 :user_invitation_visible_apps,
 :user_invitation_relationships_visible_apps,
 :bundle_id,
 :bundle_ids,
 :bundle_id_relationships_profiles,
 :bundle_id_bundle_id_capabilities,
 :bundle_id_profiles,
 :bundle_id_relationships_bundle_id_capabilities,
 :certificates,
 :certificate,
 :devices,
 :device,
 :financial_reports,
 :sales_reports,
 :profiles,
 :profile,
 :profile_bundle_id,
 :profile_relationships_bundle_id,
 :profile_relationships_certificates,
 :profile_devices,
 :profile_certificates,
 :profile_relationships_devices,
 :beta_testers,
 :beta_tester,
 :beta_tester_apps,
 :beta_tester_builds,
 :beta_tester_relationships_builds,
 :beta_tester_relationships_apps,
 :beta_tester_beta_groups,
 :beta_tester_relationships_beta_groups,
 :apps,
 :app,
 :app_beta_groups,
 :app_relationships_beta_groups,
 :app_builds,
 :app_relationships_builds,
 :app_pre_release_versions,
 :app_relationships_pre_release_versions,
 :app_beta_app_review_detail,
 :app_beta_license_agreement,
 :app_beta_app_localizations,
 :app_relationships_beta_app_review_detail,
 :app_relationships_beta_license_agreement,
 :app_relationships_beta_app_localizations,
 :pre_release_versions,
 :pre_release_version,
 :pre_release_version_relationships_app,
 :pre_release_version_builds,
 :pre_release_version_app,
 :pre_release_version_relationships_builds,
 :beta_groups,
 :beta_group,
 :beta_group_apps,
 :beta_group_relationships_app,
 :beta_group_beta_testers,
 :beta_group_relationships_builds,
 :beta_group_relationships_beta_testers,
 :builds,
 :build,
 :build_app,
 :beta_group_builds,
 :build_pre_release_version,
 :build_relationships_pre_release_version,
 :build_relationships_app,
 :build_individual_testers,
 :build_relationships_beta_app_review_submission,
 :build_relationships_individual_testers,
 :build_build_beta_detail,
 :build_relationships_build_beta_detail,
 :build_beta_app_review_submission,
 :build_relationships_app_encryption_declaration,
 :build_beta_build_localizations,
 :build_relationships_beta_build_localizations,
 :build_app_encryption_declaration,
 :app_encryption_declarations,
 :app_encryption_declaration,
 :app_encryption_declaration_app,
 :app_encryption_declaration_relationships_app,
 :build_beta_details,
 :build_beta_detail,
 :build_beta_detail_build,
 :build_beta_detail_relationships_build,
 :beta_app_localizations,
 :beta_app_localization_app,
 :beta_app_localization,
 :beta_app_localization_relationships_app,
 :beta_license_agreements,
 :beta_license_agreement_relationships_app,
 :beta_license_agreement_app,
 :beta_license_agreement,
 :beta_build_localization_relationships_build,
 :beta_build_localization_build,
 :beta_build_localizations,
 :beta_build_localization,
 :beta_app_review_details,
 :beta_app_review_detail,
 :beta_app_review_detail_relationships_app,
 :beta_app_review_detail_app,
 :beta_app_review_submissions,
 :beta_app_review_submission,
 :beta_app_review_submission_relationships_build,
 :beta_app_review_submission_build]
```

## sales_reports

```
require 'csv'
res = client.sales_reports(
  filter: {
    report_type: 'SALES',
    report_sub_type: 'SUMMARY',
    frequency: 'DAILY',
    vendor_number: '8xxx',
    report_date: Date.new(2021,3,20).strftime('%Y-%m-%d')
  }
rows = CSV.parse(hoge, col_sep: "\t", headers: true)
pry(main)> rows.entries.first.headers
=> ["Provider",
 "Provider Country",
 "SKU",
 "Developer",
 "Title",
 "Version",
 "Product Type Identifier",
 "Units",
 "Developer Proceeds",
 "Begin Date",
 "End Date",
 "Customer Currency",
 "Country Code",
 "Currency of Proceeds",
 "Apple Identifier",
 "Customer Price",
 "Promo Code",
 "Parent Identifier",
 "Subscription",
 "Period",
 "Category",
 "CMB",
 "Device",
 "Supported Platforms",
 "Proceeds Reason",
 "Preserved Pricing",
 "Client",
 "Order Type"]
```

## 





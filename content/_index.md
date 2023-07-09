---
# Leave the homepage title empty to use the site title
title:
date: 2022-10-24
type: landing

sections:
  - block: hero
    id: top
    content:
      title: Welcome to Confluent Japan Community
      image:
        filename: confluent-logo.png
      text: |-

        <!--Custom spacing-->
        <div class="mb-5"></div>
        このコミュニティはConfluentエンジニアならびにユーザー有志によるコミュニティです。ConfluentやKafkaエコシステムに関わる日本語の情報発信と共有を目的としています。(ベータ運用中)
    design:
      background:
        gradient_end: '#040531'
        gradient_start: '#14449A'
        text_color_light: true
  - block: collection
    id: blog
    content:
      title: Blogs and Announcements
      subtitle: ''
      text: ''
      # Choose how many pages you would like to display (0 = all pages)
      count: 3
      # Filter on criteria
      filters:
        folders:
          - blog
        featured_only: false
        exclude_featured: false
        exclude_future: false
        exclude_past: false
        publication_type: ""
      # Page order: descending (desc) or ascending (asc) date.
      order: desc
      archive:
        enable: true
        text: 全てのエントリ
        link: blog/
    design:
      # Choose a layout view
      view: showcase
      columns: '2'
  - block: collection
    id: talk
    content:
      title: Talks
      # Choose how many pages you would like to display (0 = all pages)
      count: 3
      filters:
        folders:
          - talk
      order: desc
      archive:
        enable: true
        text: 全てのエントリ
        link: talk/
    design:
      # Choose how many columns the section has. Valid values: '1' or '2'.
      columns: '1'
      view: showcase
      # For Showcase view, flip alternate rows?
      flip_alt_rows: true
  - block: portfolio
    id: demo
    content:
      title: Demos
      filters:
        folders:
          - demo
      # Default filter index (e.g. 0 corresponds to the first `filter_button` instance below).
      default_button_index: 0
      # Filter toolbar (optional).
      # Add or remove as many filters (`filter_button` instances) as you like.
      # To show all items, set `tag` to "*".
      # To filter by a specific tag, set `tag` to an existing tag name.
      # To remove the toolbar, delete the entire `filter_button` block.
      buttons:
        - name: All
          tag: '*'
        - name: Confluent Platform
          tag: Confluent Platform
        - name: Confluent Cloud
          tag: Confluent Cloud
    design:
      columns: '2'
      view: Card
  - block: collection
    id: publication
    content:
      title: Publications
      filters:
        folders:
          - publication
        featured_only: false
      count: 4
      archive:
        enable: true
        text: 全てのエントリ
        link: publication/
    design:
      columns: '1'
      view: showcase
      # For Showcase view, flip alternate rows?
      flip_alt_rows: true
  - block: tag_cloud
    content:
      title: Tag Cloud
      count: 20
    design:
      columns: '1'
---

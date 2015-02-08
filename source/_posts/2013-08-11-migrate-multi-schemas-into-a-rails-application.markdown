---
layout: post
title: "How to migrate a PostgreSQL DB with many schemas in a multenancy Rails application"
date: 2013-08-11 21:38
comments: true
categories: ruby, rails,migration,postgresql
---

PostgreSQL的schema是一個非常方便的功能, 適合拿來做所謂的[Multi-Tenancy](http://www.arthurtoday.com/2010/02/multi-tenant-application-multitenancy.html#.UgeXg2T08Rw)類服務; 例如無名或Pixnet這類的BSP(Blog Service Provider)或是像我的FREEBBS的免費論壇服務(Forum Service Provider), 通常稱為SAAS(Software As A Service)系統, 可以把N個相同結構的schema放在同一個資料庫裡面; 比起MySQL只能分別放在不同的資料庫裡會有很多好處。

假設今天我用PostgreSQL的schema功能來做一個BSP服務的話, DB內的schema會分成兩大類：

*  管理用schema(只有一個)
*  Blog Schema(有無數個, 每個的結構相同)

一般狀況下, 我們在使用ActiveRecord連接PosttgreSQL時可以在連接選項設定使用Schema, 像這樣：

{% codeblock  database.yml lang:yaml %}
development:
  adapter: postgresql
  encoding: unicode
  database: SHOPON_development
  pool: 5
  schema_search_path: base, blog1
{% endcodeblock %}

schema_search_path的預設值是public, 如果照以上設定, 當你執行任何migration時會先從base這個schema執行起, 也就是所有的migration都只會跑在base這個schema上面, 


但場景回到Rails/ActiveRecord上, 假如你要做一個BSP, 一般而言你有以下幾種方式：

1.  管理端(管理BSP的)和服務端(Blog服務本身)分別各自一個Rails Application目錄, 然後各有各的migrate
2.  將兩端放在同一個Rails App裡

如果用一般的方式migrate, Rails會依照你database.yml裡設定的schema_search_path的順序找第一順位來執行migrate; 這樣你要如何migrate你的資料到不同schema呢？

今天要講的就是將兩端放在同app裡的方式, 也就是

1.  在migrate檔內設定只能在服務端schema或是管理端schema執行
2.  有**單獨migrate管理端**和**單獨migrate服務端**的功能

***實現1.的方式如下***：

{% codeblock migrate_control_side.rb %}
class CreateSites < ActiveRecord::Migration
  def change
    if ActiveRecord::Base.connection.schema_search_path == 'base' #base為管理端schema
      create_table :sites do |t|
        t.string :subdn
        t.string :name
        t.timestamps
      end
    end
  end
end
{% endcodeblock %}

簡單來說就是當現在的 schema_search_path 不等於base則不執行create_table的動作; 如果是服務端, 則反過來設定schema_search_path不為base或public即可, 更進一步的可以將這兩種檢查寫成ActiveRecord::Migration的module, 像這樣：

{% codeblock multitenancy_migration.rb %}
module ActiveRecord::MultitenancyMigration
  def must_migrate_in_base #管理端
    if ActiveRecord::Base.connection.schema_search_path == 'base'
      yield
    end
  end

  def must_migrate_in_site #服務端
    schema_search_path = ActiveRecord::Base.connection.schema_search_path
    if !schema_search_path.index('base') && !schema_search_path.index('public')
      yield
    end
  end
end

ActiveRecord::Migration.class_eval do
  include ActiveRecord::MultitenancyMigration
  extend ActiveRecord::MultitenancyMigration
end
{% endcodeblock %}

由於migration檔的內部可能是以self.up(class method)或是up(instance method)的方式編寫, 因此需要同時include和extend這個模組。

***實現2.的方式如下***：

需要建立for 管理端以及 for 服務端的task 來migrate 各自的schema
{% codeblock lang:ruby base.rake%}
namespace :base do
  task migrate_db: :environment do
    ActiveRecord::Base.connection.schema_search_path = 'base'
    Rake::Task['db:migrate'].invoke
  end

  task rollback_db: :environment do
    ActiveRecord::Base.connection.schema_search_path = 'base'
    Rake::Task['db:rollback'].invoke
  end

end
{% endcodeblock %}
{%codeblock lang:ruby site.rake %}
namespace :site do

  def check_subdn_exists
    if ENV.has_key?('SUBDN')
      ActiveRecord::Base.connection.schema_search_path = ENV['SUBDN']
      yield
    else
      puts "Please assign SUBDN first!"
    end
  end


  desc 'Migrate Database of Individual site, have to assign SUBDN paramater!'
  task migrate_db: :environment do
    check_subdn_exists do
      Rake::Task['db:migrate'].invoke
    end
  end

  desc 'Rollback migration of single site'
  task rollback_db: :environment do
    check_subdn_exists do
      Rake::Task['db:rollback'].invoke
    end
  end

end
{% endcodeblock %}

簡單來說就是在migrate前先切換schema, 然後用Rake的指令呼叫原本的db:migrate/rollback出來, 當然你也可以寫一個task是去掃現存的服務端列表然後再各自migrate, 如果要寫在controller或model的話, 則必需先require rake然後再load Rakefile才行。

以上


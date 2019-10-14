---
layout: page
title: 集合框架
titlebar: 集合框架
subtitle: <span class="mega-octicon octicon-clippy"></span> &nbsp;&nbsp; 集合框架是一个类库的集合，包含实现集合的接口。接口是集合的抽象数据类型，提供对集合中所表示的内容进行单独操作的可能。
menu: collection
css: ['blog-page.css']
permalink: /collection
---

<div class="row">

    <div class="col-md-12">
    
        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='collection' or post.category=='collection' or post.keywords contains 'collection' %}
                <li class="posts-list-item">
                    <div class="posts-content">
                        <span class="posts-list-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
                        <a class="posts-list-name bubble-float-left" href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
                        <span class='circle'></span>
                    </div>
                </li>
                {% endif %}
            {% endfor %}
        </ul>
    
        <!-- Pagination -->
        {% include pagination.html %}
    
        <!-- Comments -->
       <div class="comment">
         {% include comments.html %}
       </div>
    </div>

</div>
<script>
    $(document).ready(function(){

        // Enable bootstrap tooltip
        $("body").tooltip({ selector: '[data-toggle=tooltip]' });
    
    });
</script>

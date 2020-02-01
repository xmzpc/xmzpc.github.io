---
layout: page
title: 分布式
titlebar: 分布式
subtitle: <span class="mega-octicon octicon-clippy"></span> &nbsp;&nbsp; 一个大型的系统往往被分为几个子系统来做，一个子系统可以部署在一台机器的多个 JVM 上，也可以部署在多台机器上。但是每一个系统不是独立的，不是完全独立的。需要相互通信，共同实现业务功能。
menu: distributed
css: ['blog-page.css']
permalink: /distributed
---

<div class="row">

    <div class="col-md-12">
    
        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='distributed' or post.category=='distributed' or post.keywords contains 'distributed' %}
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
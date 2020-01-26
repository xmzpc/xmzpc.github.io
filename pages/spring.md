---
layout: page
title: Spring
titlebar: spring
subtitle: <span class="mega-octicon octicon-clippy"></span> &nbsp;&nbsp; Spring 框架是一个开源的 Java 平台，它为容易而快速的开发出耐用的 Java 应用程序提供了全面的基础设施。
menu: spring
css: ['blog-page.css']
permalink: /spring
---

<div class="row">

    <div class="col-md-12">
    
        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='spring' or post.category=='spring' or post.keywords contains 'spring' %}
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
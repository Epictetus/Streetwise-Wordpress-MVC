#swpMVC

swpMVC is a lightweight MVC framework built to bring some of the experience of other
rapid application development frameworks to WordPress. Inspired largely by Rails, Express
and FuelPHP, it aims to make routing, modeling, and rendering easy, giving you more control
over your code structure than WordPress gives out of the box, without adding too much
extra work.

The simplest way to cover some of the initial concepts and get started is by examining (and using)
the starter plugin found in the starter_plugin directory.


##Starter Plugin

***

###Why a singleton?

I know a singleton is an upgraded global variable, but your plugins need to add WordPress filters and
actions. Using a singleton gives you easy access to the plugin class, and also makes sure you don't
end up running the methods you hang on your filters and actions more than once each.

###require_dependencies

This is called in the constructor, and this is where you should include your models and controllers. By
creating the plugin instance on the swp_mvc_init action, we ensure that the swpMVC core is loaded,
and the base classes which your models and controllers must extend will be available.

###add_actions

Also called in the constructor, this is used to hook the add_routes method to the swp_mvc_routes
filter, where you can add your plugin routes using the syntax described in the router section. By
placing this after the require_dependencies call in the constructor, you can be sure your controllers
are loaded when you add the routes to the system.

###add_routes

swpMVC follows a routing structure that more closely resembles Sinatra or Express than WordPress'
rewrite rules. See the next section for syntax.


##Router

***

### Adding routes

swpMVC routes are stored as an array of arrays, with each array stored representing one route using the following
structure

    <?php
    
        $route = array('controller' => 'ControllerClass',
                                'method' => 'ControllerMethod',
                                'route' => '/url/of/route/:p/:p'
                            );
        
There is no "automagic" routing, everything must be declared. This is done so that your routing structure is exactly as
you want, with no additional steps required to turn off magic routes.

### Routing parameters

Parameters in your route are represented with the token ":p"

The will be passed to your controller method in the order they are declared. Skipping named parameters allows the
framework to use [only one additional querystring variable](http://codex.wordpress.org/Rewrite_API/add_rewrite_tag)

### Auto-flush rewrite rules

The core framework will monitor whether swpMVC routes have been added, modified or removed, and flush the rewrite
rules as needed, so there is no need to do this manually.

### Example

Here's a full example of the adding routes from your swpMVC plugin based off the example plugin included in the example
directory

    <?php
        
        public function add_routes($routes)
        {
            $r[] = array('controller' => 'swpMVC_Example_Controller',
                        'method' => 'wp_style',
                        'route' => '/recent_thumbs/wp_style');
            $r[] = array('controller' => 'swpMVC_Example_Controller',
                            'method' => 'swpmvc_style',
                            'route' => '/recent_thumbs/swpmvc_style');
            $r[] = array('controller' => 'swpMVC_Example_Controller',
                            'method' => 'render_post_form',
                            'route' => '/post_form/:p');
            $s =  array_merge($routes, $r);
            return $s;
        }


##Models

***

Models must extend the swpMVCBaseModel class. This class itself extends ActiveRecord\Model, from the
[PHP ActiveRecord library](http://phpactiverecord.org). For query syntax, CRUD operations, basic model definitions, and
overloading, refer to the ActiveRecord docs. The copy included in swpMVC [includes several modifications to make it more WordPress friendly, (the diff is backwards, sorry)](https://github.com/beezee/php-activerecord/compare/master...original)
, and the swpMVCBaseModel class includes several convenience methods, all of which are covered in this
section.

###public static function tablename

Instead of declaring the model table with a static variable, we use a static method. This allows us to do the following

    <?php
        
        public static function tablename()
        {
            global $wpdb;
            return $wpdb->prefix.'posts';
        }

This model would now be multisite aware. The advantage to using a method over a variable is that we can now dynamically
define the table property for our model.

###public static function conditions

This defines any conditions that should apply to every finder query that is generated by the model. For example if I wanted
to model draft posts only:
    
    <?php
        public static function conditions()
        {
            return array("post_status = ?", "draft");
        }
        
###public static function joins

This defines any joins that should apply to every finder query that is generated by the model. In general for related eager loading,
I favor include, using the joins method only when my conditions method relies on data in another table. An example of how this
can be used to model categories:

    <?php
    
    class Category extends swpMVCBaseModel
    {
        public static function tablename()
        {
            global $wpdb;
            return $wpdb->prefix.'terms';
        }
    
        public static function conditions()
        {
            global $wpdb;
            $tt = $wpdb->prefix.'term_taxonomy';
            return array("$tt.taxonomy = ?", 'category');
        }
        
        public static function joins()
        {
            global $wpdb;
            $t = self::tablename();
            $tt = $wpdb->prefix.'term_taxonomy';
            return "LEFT JOIN $tt ON $t.term_id = $tt.term_id";
        }
    }
    
Now any finder queries generated by the Category class will include a left join on the term_taxonomy table, and
filter results to include only those where the term_taxonomy.taxonomy field has a value of "category." Filtering
subsets with models is particularly relevant in WordPress, where different "types" of data are frequently lumped
together in single tables.

One catch to using the joins method, is that calls to the [models finder methods](http://www.phpactiverecord.org/projects/main/wiki/Finders)
will need to use table prefixes for any columns that are present in both the main and joined tables. For this I recommend your
Model::tablename() methods.


###Automatic escaping

Model properties are automatically escaped when accessed directly. To override this, call the properties method on an object,
and access the properties from the resulting array.

###$model->render()

swpMVCBaseModel comes with an instance method 'render,' which accepts as an argument a Stamp view object (see [Views](#views)
section for details,) and autopopulates the Stamp using the model properties.

###public function render_{{property_name}}

These methods act as overrides for your properties when called by the render method. For example, if a Stamp view object
passed to the render method contains a tag called post_name, and your Post model has a method called render_post_name,
the return value of that method will be used to populate the Stamp object in favor of the value of the instance property
post_name.

###public function renderers

This method allows you to define additional renderers (not limited to model properties) that the model should be able to handle.
It must return an array where the keys are the Stamp tags to be replaced, and the value is an array where the first element is
the name of an instance method on the model which will return the value to be used in view population, and the second element
is an array of arguments, (an empty array if no arguments to be passed.)

For example:

    <?php
    
        public function renderers()
        {
            return array(
                'additional_render_tag' => array('additional_render_method', array())
            );
        }
        
This would replace a Stamp tag labeled 'additional_render_tag' with the result of calling $model->additional_render_method()
with no arguments.

###public static function controls

This method allows you to define form controls that should be used to interact with model properties, which will then be used by
the [render](#models/model-render) method to populate any Stamp tags of the format control\_{{property name}} and additionally
control\_label\_{{property name}} if the label attribute is defined for the control definition in the return value. The structure for the
return value on this method is again an associative array of arrays which follow the structure below:

    <?php
    
        public function controls
        {
            return array(
                'property_name' => array(
                        'type' => 'input',
                        'label' => 'Property Name',
                        'input_type' => 'button'
                    )
            );
        }
        
The following must be true of any element in the controls array for it to be valid:

*   The key must match the model property that the control corresponds to.
*   Type must be either input, select, or textarea
*   input_type is optional and only applies when type is set to input. Default is text.
*   label is optional. control\_label\_{{property name}} tags will not be replaced if no label property is defined
*   If type is select, an additional element is required with key 'options'. The value of this element must be an associative array of
    options for the dropdown, where key is the text for the option, and value is the value when that option is selected.
    
An example of a valid select control:

    <?php
    
        public function controls
        {
            return array(
                'property_name' => array(
                        'type' => 'select',
                        'label' => 'Property Name',
                        'options' => array(
                            'Option 1 Text' => 'option_1_value',
                            'Option 2 Text' => 'option_2_value'
                        )
                    )
            );
        }

When called by the $model->render() method, the generated controls will have their values set according to the values of the
model instance on which the render method was invoked, and the form will appear populated.

###Form prefixes

Each model rendering a form adds a prefix to its form elements. By default when invoked via $model->render(), the form prefix
will be the class name of the model instance invoking the render method. To override this, set the '\_prefix' property on the  model
instances form helper before calling render, as follows:

    <?php
        
        $model->form_helper()->_prefix = 'my_custom_form_prefix';
        $model->render($this->template('template_name'));
        
###Model::renderForm()

This method can be called statically on a model class to render an empty form for the class properties. The method accepts
two parameters. The first parameter is required and must be a valid Stamp view object to be populated, the second is an optional
form prefix (defaults to the class name.)

###A note about "through" relationships

Personally I've not had much success declaring [through relationships](http://www.phpactiverecord.org/projects/main/wiki/Associations#has_many_through)
with my activerecord models, so I have stuck to nesting includes during my finder calls manually. While the through relationship
does improve efficiency, I've decided it's not worth the trouble, as even without the generated joins you can typically get all
the data you will need for less than 10 queries with simple nested eager loading. Here's an example of getting 10 posts with their
related post tag and category data using simple nested eager loading:

    <?php
        
        $posts = Post::all(array('limit' => 10,
                'include' => array('postterms' =>
                    array('termtaxonomy' =>
                        array('term')
                    )
                )
            )
        );
        
If you do have some success modeling with the through relationships, please reach out via Github issues and I'll be happy to update
the docs, repo as needed.

###Included models


*       Post
*	PostMeta
*	TermRelationship
*	TermTaxonomy
*	Term
*	Comment
*	User
*	UserMeta

All models can be found in models/wordpress_models.php. There's not alot of code, and the best way to learn model definition, as
well as see what added methods are available on each model is to view the source.

##Model meta

***

the swpMVCBaseModel class includes some methods for working with meta, a popular WordPress data structure. Any table with
columns foreign_key, meta_key, meta_value will work with these methods, once you've defined a
[$has_many relationship](http://www.phpactiverecord.org/projects/main/wiki/Associations) to a model with the name meta.

###$model->meta()

This method will return an empty array if there is no $has_many [relationship](http://www.phpactiverecord.org/projects/main/wiki/Associations)
named meta defined for the model on which it is called.

If the meta relationship is defined, it must point to a meta table with columns meta_key and meta_value. By calling $model->meta()
under when these circumstances are met, the return value will be an associative array where keys are equal to meta_key, and
values are equal to the meta_value. In the case of duplicate meta_key rows for one model instance, the meta_value will be an
indexed array of all values found.

It is recommended to [eager load](http://www.phpactiverecord.org/projects/main/wiki/Finders#eager-loading) the meta when
querying for your models if you intend to use this method, to avoid the
[n+1 query problem](http://www.phabricator.com/docs/phabricator/article/Performance_N+1_Query_Problem.html)

###public function hydrate_meta

When working with ActiveRecord Models, WordPress will not serialize and unserialize data automatically for you. This method
accepts two parameters, meta_key and meta_value, and gives you the opportunity to modify meta values as necessary when
retreiving them using the $model->meta() method. (This does not apply when the $raw parameter is passed as true.)

Here's an example that will unserialize meta when the key is equal to 'serialized_meta':

    <?php
        
        public function hydrate_meta($meta_key, $meta_value)
        {
            return ($meta_key === 'serialized_meta') ?
                unserialize($meta_value) : $meta_value;
        }
        
###public function dehydrate_meta

This method serves the opposite purpose of hydrate_meta, and will be invoked on any values assigned to the 'meta_value' property
of a meta object. For a class with dehydrate_meta defined as follows:

    <?php
    
        public function dehydrate_meta($meta_key, $meta_value)
        {
            return ($meta_key === 'serialized_meta') ?
                serialize($meta_value) : $meta_value;
        }
        
Calling the below on a model instance of the class would make the following boolean statment true:

    <?php
    
        $model->meta_value = array('one', 'two', 'three');
        $model->meta_value === 'a:3:{i:0;s:3:"one";i:1;s:3:"two";i:2;s:5:"three";}';

##Views

***

swpMVC Views are based off of an older version of [Gabor DeMooij's Stamp library,](https://github.com/gabordemooij/stamp/blob/StampEngine/Stamp.php)
using extremely simple principles. Your views will contain no logic at all, and in most cases will be completely valid HTML on their own.

###Stamp tags

Stamp tags take the form of html comments, with an opening and closing comment representing one replaceable block. For example:

    <a href="<!-- url --><!-- /url -->">Link to somewhere</a>
    
gives you a replaceable region labeled url. If you place this in a file called template.tpl, and call the below code from your controller:

    <?php
        
        echo $this->template('template')->replace('url', 'http://www.somesite.com');
        
the result would be:

    <a href="http://www.somesite.com">Link to somewhere</a>
    
###Rendering models with templates

When using the [$model->render()](/#models/model-render) method, your model will automatically replace tags named according
to the following conventions:

    <!-- attribute\_name --><!-- /attribute\_name -->

*  Gets replaced with a model property named attribute\_name, or the return value of model instance method
    render\_attribute\_name if such method exists.
    
    <!-- control\_attribute\_name --><!-- /control\_attribute\_name -->
    
*  Gets replaced with the 'attribute\_name' element of the array returned by static class method [controls](#models/public-static-function-controls)

    <!-- control\_label\_attribute\_name --><!-- /control\_label\_attribute\_name -->
    
*  Gets replaced with the value of the 'label' key under the 'attribute\_name' key of the array returned by static class method controls

##Controllers

***

##Logging

***
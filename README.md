# Géolocalisation de produits 

## Introduction

Ce dépôt est un tutoriel qui explique comment utiliser l'API de son site [Kiubi](http://www.kiubi.com) pour afficher une liste de produits sur une carte Google Maps avec la possibilité d'ajouter ces produits au panier directement à partir de la carte.

## Prérequis

Ce tutoriel suppose que vous avez un site Kiubi et qu'il est bien configuré : 

 - l'API est activée
 - le catalogue et l'ecommerce sont activés 
 - le site est en thème personnalisé, basé sur Shiroi

Il est préférable d'être à l'aise avec la manipulation des thèmes personnalisés. En cas de besoin, le [guide du designer](http://doc.kiubi.com) est là.

Ce tutoriel est applicable à tout thème graphique mais les exemples de codes donnés sont optimisés pour un rendu basé sur le thème Shiroi.

## Afficher des produits sur une Google Map

La liste des produits va être remplacée par une carte Google Maps sur laquelle chaque produit sera représenté par un marqueur situé à l'endroit défini sur la fiche produit dans le backoffice.

Un clic sur un marqueur affichera une bulle comportant le nom du produit ainsi que son image principale et une courte description, et si le produit est disponible il sera possible de l'ajouter au panier en un clic.

### Mise en place

#### Modification du type de produit

Copiez le fichier de ce dépôt `theme/fr/produits/simple/config.xml` ou modifiez le fichier existant dans votre thème graphique pour définir les champs `latitude` et `longitude` au produit :

<pre lang="xml">
&lt;?xml version="1.0" encoding="iso-8859-1"?>
&lt;!-- 
Configuration du produit "Produit simple"
-->
&lt;!DOCTYPE type SYSTEM "http://www.kiubi-admin.com/DTD/typesproduits.dtd"> 
	&lt;type tri="1">
		&lt;desc>Produit simple&lt;/desc>
		&lt;listechamps>
			&lt;champ type="text" champ="texte11" intitule="Latitude"/>
			&lt;champ type="text" champ="texte12" intitule="Longitude"/>
		&lt;/listechamps>
	&lt;/type>
</pre>

#### Inclusions des bibliothèques javascript

Pour réaliser notre carte de produits il nous faut 3 librairies, le client JS Kiubi pour utiliser l’API, l’API Google Maps pour l’affichage de la carte ainsi qu’un plugin jQuery qui nous permettra de faciliter l’intégration des éléments sur la carte.

Le plugin jQuery utilisé est Gmap3 et est distribué sous licence GPL v3. Les sources sont disponibles à l’adresse suivante : [http://gmap3.net/](http://gmap3.net/) 
On rajoute l’inclusion des trois fichiers grâce au module `Injection de code` et on met le code d’inclusion avant la balise `</head>` :

<pre lang="html">
&lt;script type="text/javascript" src="{cdn}/js/kiubi.api.pfo.jquery-1.0.min.js">&lt;/script>
&lt;script type="text/javascript" src="http://maps.google.com/maps/api/js?sensor=false&amp;amp;language=fr">&lt;/script>
&lt;script type="text/javascript" src="//cdn.jsdelivr.net/gmap3/5.1.1/gmap3.min.js">&lt;/script>
</pre>

*Nous utilisons ici un cdn public tiers pour l’inclusion du plugin jQuery, vous pouvez également récupérer le fichier et le déposer dans votre thème puis l’inclure dans vos templates de pages.*

#### Widget liste des produits

Pour ajouter la carte contenant la liste de nos produits dans le widget `Liste des produits`, on remplace le template `theme/fr/widget/catalog/liste_produit/index.html` du thème par celui de ce dépôt.

Allez sur la page du catalogue de votre site, une carte s'affiche avec les produits du catalogue chacun représenté par un marqueur sur la carte. Un clic sur un marqueur ouvre une bulle avec les informations sur le produit. Si le produit est disponible, un bouton d'ajout au panier est affiché et un clic sur ce bouton ajoute directement le produit au panier. 

### Explications

Examinons en détail le code HTML du template `liste_produit/index.html` :

La fonction `showMap()` va réaliser l’essentiel du travail en affichant la carte, récupérant les produits pour créer des points géolocalisés, puis permettre l’ajout au panier en un clic.
Les premières lignes de cette fonction masque la liste par défaut des produits, et adapte certains styles de la page pour optimiser l’affichage de la carte.
On limitera le nombre de marqueurs sur la carte en fonction de la configuration du widget sur le nombre de produits à afficher : `{widget_limit}`

<pre lang="javascript">
var showMap = function() {
	var nb_markers = 0, max_markers = parseInt("{widget_limit}");
	$('section.section').css('width', '100%');
	$('#page').css('max-width', '100%');	
	$('#main').data('initfloat', $('#main').css('float'));
	$('#main').css('width', 'auto').css('float', 'none').css('margin', 0);
	$('.header_inner').css('padding', 0);
	$('.copyright').css('margin', 0);
	$('.copyright p').css('display', 'inline-block');
	$('footer').css('padding', 0).css('border', 'none');
	$('header#top .header_inner .menu_header').css('margin', 0).css('border', 'none');
	$('header#top h2').css({margin:0, position: 'absolute', top: '5px', 'font-size': '1.8em'});
	$('#liste-init,.chemin,#sub_header,#top h3,#top form,.sidebar,.logo,.menu_header time,footer .footer_inner,#message').hide();
</pre>

Le code suivant va créer et afficher la carte Google Maps. Cette carte va prendre 100% de la largeur de la fenêtre du navigateur et va contenir un lien fixe en haut de la carte permettant de revenir à l’affichage par défaut.
<pre lang="javascript">
$('#gmap').css('width', '100%').css('height', $(document).height()-$('header').height()-$('footer').height()).show().gmap3({
		map:{
			options:{
				zoom:6,
				scrollwheel: false
			}
		},
		panel:{
			options:{
			  content: '&lt;a id="map-backto-list" href="javascript:;">Affichage classique de la liste des produits&lt;/a>',
			  bottom: 80,
			  left: 0
			}
		  }
	});	</pre>

En fonction de la configuration du widget, nous allons requêter via l’API la liste des produits du catalogue ou de la catégorie sélectionnée, tout en reprenant l’ensemble des paramètres de filtrages définis dans le widget.
Une fois la requête terminée, nous ajoutons les points sur la carte avec la méthode `addMarkers()` puis enfin si les résultats sont paginés et qu’une autre page existe, et que notre nombre de points n’a pas atteint sa limite, on récupère la page suivante de produits.
<pre lang="javascript">
var xtra = {
	'limit'        : parseInt("{widget_limit}"),
	'sort'         : "{widget_order_api}",
	'available'    : "{widget_dispo}",
	'in_stock'     : "{widget_stock}",
	'tags'         : "{widget_tags}",
	'tags_rule'    : ("{widget_tags_logique}" == "et" ? "and" : "or"),
	'extra_fields' : "texts,thumb,price_label,variants"
};
if('{widget_categorie_id}' == '') {		
	var q = kiubi.catalog.getProducts(xtra);
} else {
	var q = kiubi.catalog.getProductsByCategory(parseInt('{widget_categorie_id}'), xtra);
}
q.done(function(meta, data){
	addMarkers(data);
	if(kiubi.hasNextPage(meta) && nb_markers&lt;max_markers){
		getNextPage(meta);
	}
});
</pre>

La méthode `getNextPage()` récupère la page suivante de notre requête de produits et ajoute les points sur la carte. Cette méthode s’appelle elle même s’il y a encore une page suivante dans les résultats et si le nombre de points sur la carte n’a pas atteint le nombre maximal défini.
<pre lang="javascript">
var getNextPage = function(meta) {
	var d = kiubi.getNextPage(meta);
	d.done(function(meta, data){
		addMarkers(data);
		if(kiubi.hasNextPage(meta) && nb_markers&lt;max_markers){
			getNextPage(meta);
		}
	});
};
</pre>

La méthode suivante ajoute les points sur la carte. Pour chaque produit, on vérifie s’il est bien géolocalisé, sinon on passe au suivant puis on définit le contenu html de l’infobulle qui sera affiché lors du clic sur le marqueur. Dans l’infobulle on affiche le titre du produit, son image principale, sa courte description et la liste des variantes.
Ensuite on ajoute un marqueur sur notre carte en lui précisant sa géolocalisation et son contenu que nous venons de créer. Enfin, on se greffe sur l’événement `click` du point afin de créer ou réafficher l’infobulle contenant les informations.
<pre lang="javascript">
var addMarkers = function(data) {
	for(var i=0; i&lt;data.length; i++) {
		if(!data[i].text11 || !data[i].text12 || nb_markers>=max_markers) continue;
		nb_markers++;
		
		var html = '&lt;div style="width:480px;height:320px">';
		html += '&lt;h2>'+data[i].name+'&lt;/h2>';
		html += '&lt;img style="float:left;margin-right:10px;" src="'+data[i].main_thumb.url_g_miniature+'"/>';				
		html += '&lt;div>'+data[i].header+'&lt;/div>&lt;br/>';
		if(data[i].variants.length&lt;2) {
			html += '&lt;p class="prix" style="color: #A6A6A6;font-size: 1.2em;">'+data[i].variants[0].price_inc_vat_label;
			if(data[i].is_discounted)
				html += ' &lt;del style="color: #DF3F52;font-size: 0.8em;">'+data[i].price_base_inc_vat_label+'&lt;/del>';
			html += '&lt;/p>&lt;br/>';
		}
		html += '&lt;select class="p-info" '+(data[i].variants.length&lt;2?' style="display:none;"':'')+'>';
		for(var v=0; v&lt;data[i].variants.length; v++) {
			var va = data[i].variants[v];
			html += '&lt;option value="'+va.id+'" data-available="'+(va.is_available && va.in_stock)+'">'+va.name+' : '+va.price_inc_vat_label+'&lt;/option>';
		}
		html += '&lt;/select>';
		html += '&lt;/div>';
		$('#gmap').gmap3({
			marker:{
				values:[
				  {latLng:[data[i].text11, data[i].text12], data:html},				  
				],
				events:{
					click: function(marker, event, context){
					var map = $(this).gmap3("get"),
					infowindow = $(this).gmap3({get:{name:"infowindow"}});
					if (infowindow){
					infowindow.open(map, marker);
						infowindow.setContent(context.data);
					} else {
						$(this).gmap3({
							infowindow:{
							anchor:marker,
							options:{content: context.data},
							events:{
								domready: function(){
									$(this).find('.p-nfo').change();
									}
								}
							}
						});
					}
					$('.p-info').change();
				}
			}				
		});
	}
};
</pre>

Le code suivant se greffe sur l’événement de changement de valeur des listes de variantes dans les infobulles. On repère les éléments avec leur classe css `p-info`
Si la variante sélectionnée est disponible, on ajoute un bouton d’ajout au panier, sinon on indique que l’article n’est pas disponible.
<pre lang="javascript">
$(document).on('change', '.p-info', function(){
	$(this).parent().find('input,p:not(.prix)').remove();
	if($(this).find(':selected').data('available')==true) {
		$(this).after('&lt;input type="submit" class="bouton map-addtocart" value="Ajouter au panier"/>');
	} else {
		$(this).after('&lt;p class="alerte">Article non disponible&lt;/p>');
	}
});
</pre>

Le code suivant se greffe sur l’événement `click` des boutons ajouter au panier dans les infobulles. On repère les éléments avec leur classe css `map-addtocart`
On vérifie si la variante est bien disponible avant de l’ajouter au panier, puis on appelle l’API pour effectuer l’opération : `kiubi.cart.addItem()` 
Si le produit a correctement été ajouté au panier, la méthode `done()` est appelé, elle affiche un message à l’utilisateur, puis met à jour le nombre de produits et le montant du panier dans l’encart en haut à droite de la page.
En cas d’erreur à l’ajout, la méthode `fail()` est appelée et affiche le message d’erreur à l’utilisateur.

<pre lang="javascript">
$(document).on('click', '.map-addtocart', function(){
	$(this).parent().find('p:not(.prix)').remove();
	if($(this).prev('select').find(':selected').data('available')==true) {
		var me = $(this);
		var c = kiubi.cart.addItem($(this).prev('select').val(), '+1', null, {extra_fields: 'price_label'});
		c.done(function(meta, data){
			me.after('&lt;p class="alerte">Produit ajout&eacute; au panier !&lt;/p>');
			$('.panier_link div').remove();
			$('.panier_link a').html(data.cart.items_count+' article'+(data.cart.items_count>1?'s':'')+' - '+data.cart.price_total_inc_vat_label);
		});
		c.fail(function(meta, error, data){
			me.after('&lt;p class="erreur">'+error.message+'&lt;/p>');
		});
	}
});
</pre>

Le code suivant est appelé au clic du lien statique sur la carte pour revenir à la liste normale. Cela supprime la carte, rétablit les styles de la page qui avaient été modifiés pour la carte et réaffiche la liste des produits.
<pre lang="javascript">
$(document).on('click', '#map-backto-list', function(){
	$('#gmap').gmap3('destroy').hide();
	$('section.section').css('width', '980px');
	$('#page').css('max-width', '1060px');	
	$('#main').css('float', $('#main').data('initfloat'));
	$('#main').css('width', '').css('margin-bottom', '30px');
	$('.copyright').css('margin', '0 auto 20px 0');
	$('.copyright p').css('display', 'block');
	$('footer').css('padding', '20px 0').css('border-top', '1px solid #CCCCCC');
	$('header#top .header_inner .menu_header').css('margin', '0 0 40px').css('border-bottom', '1px solid #DDDDDD');
	$('header#top h2').css({margin: '15px 0', position: 'relative', top: '', 'font-size': '3em'});
	$('.header_inner').css('padding', '0 0 40px 0');
	$('#liste-init,.chemin,#sub_header,#top h3,#top form,.sidebar,.logo,.menu_header time,footer .footer_inner,#message').show();
});
</pre>

Au chargement de la page, on masque la liste des produits, on ajoute un lien pour relancer l’affichage de la carte dans le cas où on revient à l’affichage par défaut, puis on affiche la carte en appelant `showMap()`. On se greffe également sur l'événement `resize` afin de réactualiser la carte et garder notre hauteur dynamique.
<pre lang="javascript">
$(function(){
	$('#liste-init').hide();
	$('article.produits > p:first').append(' | &lt;a href="javascript:showMap();">Afficher les produits sur une carte&lt;/a>');
	showMap();
	$(window).resize(function(){
		$('#gmap').gmap3('destroy').hide();
		showMap();
	});
});
</pre>

Et voici les seules modifications html du template d’origine. On ajoute un conteneur pour la carte, et on englobe le reste du template par défaut dans un autre conteneur qui permet de masquer la liste lorsque la carte est affichée.
<pre lang="html">
&lt;div id="gmap">&lt;/div>
&lt;div id="liste-init">
...
&lt;/div>
</pre>

### Exemple complet

<pre lang="html">
&lt;!-- 
Template du Widget "Liste des produits"
-->

&lt;!-- BEGIN: main -->
&lt;style>
section #gmap img { max-width: none; }
&lt;/style>
&lt;script type="text/javascript">
var showMap = function() {
	var nb_markers = 0, max_markers = parseInt("{widget_limit}");
	$('section.section').css('width', '100%');
	$('#page').css('max-width', '100%');	
	$('#main').data('initfloat', $('#main').css('float'));
	$('#main').css('width', 'auto').css('float', 'none').css('margin', 0);
	$('.header_inner').css('padding', 0);
	$('.copyright').css('margin', 0);
	$('.copyright p').css('display', 'inline-block');
	$('footer').css('padding', 0).css('border', 'none');
	$('header#top .header_inner .menu_header').css('margin', 0).css('border', 'none');
	$('header#top h2').css({margin:0, position: 'absolute', top: '5px', 'font-size': '1.8em'});
	$('#liste-init,.chemin,#sub_header,#top h3,#top form,.sidebar,.logo,.menu_header time,footer .footer_inner,#message').hide();
	$('#gmap').css('width', '100%').css('height', $(document).height()-$('header').height()-$('footer').height()).show().gmap3({
		map:{
			options:{
				zoom:6,
				scrollwheel: false
			}
		},
		panel:{
			options:{
			  content: '&lt;a id="map-backto-list" href="javascript:;">Affichage classique de la liste des produits&lt;/a>',
			  bottom: 80,
			  left: 0
			}
		  }
	});	
	$('#map-backto-list').css({'background-color':'#fff', color:'#000', padding:'10px 15px' }).parent().css('margin-top', '30px');
	var xtra = {
		'limit'        : parseInt("{widget_limit}"),
		'sort'         : "{widget_order_api}",
		'available'    : "{widget_dispo}",
		'in_stock'     : "{widget_stock}",
		'tags'         : "{widget_tags}",
		'tags_rule'    : ("{widget_tags_logique}" == "et" ? "and" : "or"),
		'extra_fields' : "texts,thumb,price_label,variants"
	};
	if('{widget_categorie_id}' == '') {		
		var q = kiubi.catalog.getProducts(xtra);
	} else {
		var q = kiubi.catalog.getProductsByCategory(parseInt('{widget_categorie_id}'), xtra);
	}
	q.done(function(meta, data){
		addMarkers(data);
		if(kiubi.hasNextPage(meta) && nb_markers&lt;max_markers){
			getNextPage(meta);
		}
	});
	var getNextPage = function(meta) {
		var d = kiubi.getNextPage(meta);
		d.done(function(meta, data){
			addMarkers(data);
			if(kiubi.hasNextPage(meta) && nb_markers&lt;max_markers){
				getNextPage(meta);
			}
		});
	};
	var addMarkers = function(data) {
		for(var i=0; i&lt;data.length; i++) {
			if(!data[i].text11 || !data[i].text12 || nb_markers>=max_markers) continue;
			nb_markers++;
			
			var html = '&lt;div style="width:480px;height:320px">';
			html += '&lt;h2>'+data[i].name+'&lt;/h2>';
			html += '&lt;img style="float:left;margin-right:10px;" src="'+data[i].main_thumb.url_g_miniature+'"/>';				
			html += '&lt;div>'+data[i].header+'&lt;/div>&lt;br/>';
			if(data[i].variants.length&lt;2) {
				html += '&lt;p class="prix" style="color: #A6A6A6;font-size: 1.2em;">'+data[i].variants[0].price_inc_vat_label;
				if(data[i].is_discounted)
					html += ' &lt;del style="color: #DF3F52;font-size: 0.8em;">'+data[i].price_base_inc_vat_label+'&lt;/del>';
				html += '&lt;/p>&lt;br/>';
			}
			html += '&lt;select class="p-info" '+(data[i].variants.length&lt;2?' style="display:none;"':'')+'>';
			for(var v=0; v&lt;data[i].variants.length; v++) {
				var va = data[i].variants[v];
				html += '&lt;option value="'+va.id+'" data-available="'+(va.is_available && va.in_stock)+'">'+va.name+' : '+va.price_inc_vat_label+'&lt;/option>';
			}
			html += '&lt;/select>';
			html += '&lt;/div>';
			$('#gmap').gmap3({
				marker:{
					values:[
					  {latLng:[data[i].text11, data[i].text12], data:html}					  
					],
					events:{
						click: function(marker, event, context){
						  var map = $(this).gmap3("get"),
							infowindow = $(this).gmap3({get:{name:"infowindow"}});
						  if (infowindow){
							infowindow.open(map, marker);
							infowindow.setContent(context.data);
						  } else {
							$(this).gmap3({
							  infowindow:{
								anchor:marker,
								options:{content: context.data},
								events:{
									domready: function(){
										$(this).find('.p-info').change();
									}
								}
							  }
							});
						  }
						  $('.p-info').change();
						}
					}
				}
			});
		}
	};		
};
$(document).on('change', '.p-info', function(){
	$(this).parent().find('input,p:not(.prix)').remove();
	if($(this).find(':selected').data('available')==true) {
		$(this).after('&lt;input type="submit" class="bouton map-addtocart" value="Ajouter au panier"/>');
	} else {
		$(this).after('&lt;p class="alerte">Article non disponible&lt;/p>');
	}
});

$(document).on('click', '.map-addtocart', function(){
	$(this).parent().find('p:not(.prix)').remove();
	if($(this).prev('select').find(':selected').data('available')==true) {
		var me = $(this);
		var c = kiubi.cart.addItem($(this).prev('select').val(), '+1', null, {extra_fields: 'price_label'});
		c.done(function(meta, data){
			me.after('&lt;p class="alerte">Produit ajout&eacute; au panier !&lt;/p>');
			$('.panier_link div').remove();
			$('.panier_link a').html(data.cart.items_count+' article'+(data.cart.items_count>1?'s':'')+' - '+data.cart.price_total_inc_vat_label);
		});
		c.fail(function(meta, error, data){
			me.after('&lt;p class="erreur">'+error.message+'&lt;/p>');
		});
	}
});

$(document).on('click', '#map-backto-list', function(){
	$('#gmap').gmap3('destroy').hide();
	$('section.section').css('width', '980px');
	$('#page').css('max-width', '1060px');	
	$('#main').css('float', $('#main').data('initfloat'));
	$('#main').css('width', '').css('margin-bottom', '30px');
	$('.copyright').css('margin', '0 auto 20px 0');
	$('.copyright p').css('display', 'block');
	$('footer').css('padding', '20px 0').css('border-top', '1px solid #CCCCCC');
	$('header#top .header_inner .menu_header').css('margin', '0 0 40px').css('border-bottom', '1px solid #DDDDDD');
	$('header#top h2').css({margin: '15px 0', position: 'relative', top: '', 'font-size': '3em'});
	$('.header_inner').css('padding', '0 0 40px 0');
	$('#liste-init,.chemin,#sub_header,#top h3,#top form,.sidebar,.logo,.menu_header time,footer .footer_inner,#message').show();
});

$(function(){
	$('#liste-init').hide();
	$('article.produits > p:first').append(' | &lt;a href="javascript:showMap();">Afficher les produits sur une carte&lt;/a>');
	showMap();	
	$(window).resize(function(){
		$('#gmap').gmap3('destroy').hide();
		showMap();
	});
});
&lt;/script>
&lt;div id="gmap">&lt;/div>
&lt;div id="liste-init">
&lt;article class="produits">
  &lt;!-- BEGIN:intitule -->
  &lt;header class="post_header">
    &lt;h1>{intitule}&lt;/h1>
  &lt;/header>
  &lt;!-- END:intitule -->
  &lt;p>&lt;a href="{lien_affichage}=l">Affichage en liste&lt;/a> | &lt;a href="{lien_affichage}=v">Affichage en vignette&lt;/a>&lt;/p>
  &lt;div class="tri">
    &lt;select name="select" id="select" onchange="window.location.href=this.options[this.selectedIndex].value">
      &lt;option value="" >Trier par...&lt;/option>
      &lt;option value="{lien_tri}=po">Produit&lt;/option>
      &lt;option value="{lien_tri}=pi">Prix&lt;/option>
      &lt;option value="{lien_tri}=n">Note&lt;/option>
      &lt;option value="{lien_tri}=d">Date de disponibilit&eacute;&lt;/option>
    &lt;/select>
  &lt;/div>
  {liste_produits}
&lt;/article>
  &lt;!-- BEGIN: nav2 -->
  &lt;div class="nav">
    &lt;!-- BEGIN: premier -->
    &lt;a href="{lien_premier}" title="premi&egrave;re page">premi&egrave;re page&lt;/a>
    &lt;!-- END: premier -->
    &lt;!-- BEGIN: precedent -->
    &lt;a href="{lien_precedent}" title="page pr&eacute;c&eacute;dente">page pr&eacute;c&eacute;dente&lt;/a>
    &lt;!-- END: precedent -->
    &lt;!-- BEGIN: pages -->
    &lt;a href="{lien_page}" class="{selected}">{page}&lt;/a>
    &lt;!-- END: pages -->
    &lt;!-- BEGIN: suivant -->
    &lt;a href="{lien_suivant}" title="page suivante">page suivante&lt;/a>
    &lt;!-- END: suivant -->
    &lt;!-- BEGIN: dernier -->
    &lt;a href="{lien_dernier}" title="derni&egrave;re page">derni&egrave;re page&lt;/a>
    &lt;!-- END: dernier -->
  &lt;/div>
  &lt;!-- END: nav2 -->
&lt;/div>
&lt;!-- END: main -->
</pre>

## Afficher les produits du panier sur une Google Map

Au dessus de la liste des produits du panier, une carte Google Maps est affichée. Elle comporte un marqueur pour chaque produit du panier. Le titre, l'image et la description du produit sont affichés dans une infobulle lors d'un clic sur chaque marqueur.

### Mise en place

Dans la page du détail panier nous allons afficher une carte avec les produits présents dans le panier. Pour ce faire nous allons modifier le template du widget détail panier : `theme/fr/widget/commandes/panier_detail/index.html` avec celui présent dans ce dépôt.

Ajoutez un ou plusieurs produits dans votre panier, la page du détail panier comporte une carte avec autant de marqueurs que de produits dans le panier. Chaque marqueur peut être cliquer et une infobulle s'affiche avec les informations du produit.

### Explications 

Au chargement de la page nous récupérons le contenu du panier via l'API, puis affichons une carte Google Maps
<pre lang="javascript">
$(function(){
	kiubi.cart.get().done(function(meta, data){
		$('#gmap').css('width', '100%').css('height', '300px').show().gmap3({
			map:{
				options:{
					zoom:5,
					scrollwheel: false
				}
			}		
		});
</pre>

Nous parcourons ensuite chaque item du panier et récupérons le détail produit via l'API afin de connaitre sa latitude et longitude via les champs `text11` et `text12`, si le produit n'est pas géolocalisé nous n'en tenons pas compte :
<pre lang="javascript">
for(var i=0; i&lt;data.items.length; i++) {
	kiubi.catalog.getProduct(data.items[i].product_id, {extra_fields:'texts'}).done(function(meta, data){
		if(!data.text11 || !data.text12) return;
</pre>


Enfin, on construit chaque point sur la carte avec son infobulle contenant son nom, son image et sa courte description :
<pre lang="javascript">
var html = '&lt;div style="width:360px;height:150px">';
html += '&lt;a href="'+data.url+'">&lt;h2>'+data.name+'&lt;/h2>&lt;/a>';
html += '&lt;img style="float:left;margin-right:10px;" src="'+data.main_thumb.url_miniature+'"/>';
html += '&lt;div>'+data.header+'&lt;/div>&lt;br/>';
html += '&lt;/div>';
$('#gmap').gmap3({
	marker:{
		values:[
		  {latLng:[data.text11, data.text12], data:html}		],
	events:{
		click: function(marker, event, context){
					var map = $(this).gmap3("get"),
					infowindow = $(this).gmap3({get:{name:"infowindow"}});
					if (infowindow){
						infowindow.open(map, marker);
						infowindow.setContent(context.data);
					} else {
						$(this).gmap3({
					    infowindow:{
							anchor:marker,
							options:{content: context.data}
						}
					});
				}
		}
	}
}
</pre>

Enfin, on ajoute le conteneur html pour notre carte :
<pre lang="html">
&lt;div id="gmap">&lt;/div>
</pre>

### Exemple complet

<pre lang="html">
&lt;!-- 
Template du Widget "Détail du panier"
-->
&lt;!-- BEGIN:main -->
&lt;style>
section #gmap img { max-width: none; }
&lt;/style>
&lt;ul class="chemin_commande">
  &lt;li>&lt;a href="{baseLangue}/ecommerce/panier.html" title="Panier" class="actif">1. Panier&lt;/a>&lt;/li>
  &lt;li>2. Authentification&lt;/li>
  &lt;li>3. Livraison&lt;/li>
  &lt;li>4. Mode de paiement&lt;/li>
  &lt;li>5. Validation&lt;/li>
&lt;/ul>
&lt;article class="panier_detail">
  &lt;!-- BEGIN:intitule -->
  &lt;h1>{intitule}&lt;/h1>
  &lt;!-- END:intitule -->
  &lt;script type="text/javascript">
	$(function(){
		kiubi.cart.get().done(function(meta, data){
			$('#gmap').css('width', '100%').css('height', '300px').show().gmap3({
				map:{
					options:{
						zoom:5,
						scrollwheel: false
					}
				}		
			});
			for(var i=0; i&lt;data.items.length; i++) {
				kiubi.catalog.getProduct(data.items[i].product_id, {extra_fields:'texts'}).done(function(meta, data){						
					if(!data.text11 || !data.text12) return;

					var html = '&lt;div style="width:360px;height:150px">';
					html += '&lt;a href="'+data.url+'">&lt;h2>'+data.name+'&lt;/h2>&lt;/a>';
					html += '&lt;img style="float:left;margin-right:10px;" src="'+data.main_thumb.url_miniature+'"/>';	
					html += '&lt;div>'+data.header+'&lt;/div>&lt;br/>';
					html += '&lt;/div>';
					$('#gmap').gmap3({
						marker:{
							values:[
							  {latLng:[data.text11, data.text12], data:html}					  
							],
							events:{
								click: function(marker, event, context){
								  var map = $(this).gmap3("get"),
									infowindow = $(this).gmap3({get:{name:"infowindow"}});
								  if (infowindow){
									infowindow.open(map, marker);
									infowindow.setContent(context.data);
								  } else {
									$(this).gmap3({
									  infowindow:{
										anchor:marker,
										options:{content: context.data}
									  }
									});
								  }
								}
							}
						}
					});
				});
			}
		});
	});
	&lt;/script>
	&lt;div id="gmap">&lt;/div>
  &lt;!-- BEGIN:produits -->
  &lt;form method="post" action="">
    &lt;table class="table_commande">
      &lt;tr>
        &lt;th colspan="2" scope="col" style="width: 100%; text-align: left;">{nb_produits} article{pluriel_produits}&lt;/th>
        &lt;th nowrap="nowrap" scope="col">Prix&lt;/th>
        &lt;th scope="col" style="text-align: center;">Qte&lt;/th>
        &lt;th nowrap="nowrap" scope="col">Total&lt;/th>
        &lt;th scope="col" class="produit_supp">&nbsp;&lt;/th>
      &lt;/tr>
      &lt;!-- BEGIN:produit -->
      &lt;tr>
        &lt;td style="width: {miniature_l}px;" class="produit_illustration">&lt;!-- BEGIN:illustration -->
          &lt;figure>&lt;a href="{lien_produit}" title="{intitule_produit|htmlentities} - Cliquez pour acc&eacute;der">&lt;img src="{racine}/media/miniature/{illustration}" alt="{intitule_produit|htmlentities}" class="illustration" />&lt;/a>&lt;/figure>
          &lt;!-- END:illustration -->
          &lt;!-- BEGIN: noillustration -->
          &lt;figure>&lt;a href="{lien_produit}" title="{intitule_produit|htmlentities} - Cliquez pour acc&eacute;der">&lt;img src="{racine}/{theme}/fr/images/produit_mini.gif" alt="{intitule_produit|htmlentities}" class="illustration" style="width: {miniature_l}px; height: {miniature_h}px;" />&lt;/a>&lt;/figure>
          &lt;!-- END: noillustration -->&lt;/td>
        &lt;td class="produit_info">&lt;strong>&lt;a href="{lien_produit}">{intitule_produit}&lt;/a>&lt;/strong>&lt;br />
          &lt;!-- BEGIN:reference -->
          &lt;p class="ref">Ref. : {reference}&lt;/p>
          &lt;!-- END:reference -->
          &lt;p class="variante">{intitule_variante}&lt;/p>
          &lt;!-- BEGIN:accroche -->
          &lt;p class="desc">{accroche}&lt;/p>
          &lt;!-- END:accroche -->&lt;/td>
        &lt;td align="right" class="produit_pu">{prix_unitaire}&lt;/td>
        &lt;td nowrap="nowrap" class="produit_qte">&lt;input name="qt[{ref}]" type="text" class="textfield" value="{qt}" style="width:30px" />
          &lt;!-- BEGIN:erreur_qt -->
          &lt;div class="erreurs">max. : {max}&lt;/div>
          &lt;!-- END:erreur_qt -->&lt;/td>
        &lt;td align="right" class="produit_tt">{prix_total}&lt;/td>
        &lt;td class="produit_supp">&lt;a href="{lien_supprimer}" title="Supprimer du panier">Supprimer&lt;/a>&lt;/td>
      &lt;/tr>
      &lt;!-- END:produit -->
      &lt;!-- BEGIN:option -->
      &lt;tr>
		&lt;!-- BEGIN:simple -->
        &lt;td colspan="2" style="text-align: left;">{intitule_option} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
		&lt;!-- BEGIN:offerte -->
		&lt;td>&nbsp;&lt;/td>
		&lt;td style="text-align: center;">{qt}&lt;/td>
		&lt;td>Offert&lt;/td>
		&lt;!-- END:offerte -->
		&lt;!-- BEGIN:payante -->
		&lt;td>{prix_unitaire}&lt;/td>
		&lt;td style="text-align: center;">{qt}&lt;/td>
        &lt;td>{prix_total}&lt;/td>
		&lt;!-- END:payante -->
		&lt;!-- END:simple -->
		&lt;!-- BEGIN:textarea -->
        &lt;td colspan="2" style="text-align: left;">{intitule_option} : {valeur_option} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
		&lt;!-- BEGIN:offerte -->
		&lt;td>&nbsp;&lt;/td>
		&lt;td style="text-align: center;">{qt}&lt;/td>
		&lt;td>Offert&lt;/td>
		&lt;!-- END:offerte -->
		&lt;!-- BEGIN:payante -->
		&lt;td>{prix_unitaire}&lt;/td>
		&lt;td style="text-align: center;">{qt}&lt;/td>
        &lt;td>{prix_total}&lt;/td>
		&lt;!-- END:payante -->
		&lt;!-- END:textarea -->
		&lt;!-- BEGIN:select -->
        &lt;td colspan="2" style="text-align: left;">{intitule_option} : {valeur_option} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
		&lt;!-- BEGIN:offerte -->
		&lt;td>&nbsp;&lt;/td>
		&lt;td style="text-align: center;">{qt}&lt;/td>
		&lt;td>Offert&lt;/td>
		&lt;!-- END:offerte -->
		&lt;!-- BEGIN:payante -->
		&lt;td>{prix_unitaire}&lt;/td>
		&lt;td style="text-align: center;">{qt}&lt;/td>
        &lt;td>{prix_total}&lt;/td>
		&lt;!-- END:payante -->
		&lt;!-- END:select -->
        &lt;td class="produit_supp">&lt;a href="{lien_supprimer}" title="Supprimer du panier">Supprimer&lt;/a>&lt;/td>
      &lt;/tr>
      &lt;!-- END:option -->
      &lt;!-- BEGIN:bon -->
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">{intitule_produit} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
        &lt;td class="produit_tt">{prix_total}&lt;/td>
        &lt;td class="produit_supp">&lt;a href="{lien_supprimer}" title="Supprimer du panier">Supprimer&lt;/a>&lt;/td>
      &lt;/tr>
      &lt;!-- END:bon -->
      &lt;!-- BEGIN:remise -->
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">{intitule_remise} &lt;a href="{lien_supprimer}" title="Supprimer du panier" class="bt_supp">Supprimer&lt;/a>&lt;/td>
        &lt;td align="right">&nbsp;&lt;/td>
        &lt;td class="produit_supp">&lt;a href="{lien_supprimer}" title="Supprimer du panier">Supprimer&lt;/a>&lt;/td>
      &lt;/tr>
      &lt;!-- END:remise -->
      &lt;tr class="tva">
        &lt;td colspan="4" style="text-align: left;">&lt;strong>Total hors frais de livraison&lt;/strong>&lt;/td>
        &lt;td align="right">&lt;strong>{total_articles}&lt;/strong>&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;!-- BEGIN:livrable -->
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">Frais de port applicables pour les livraisons vers : {zone_livraison}&lt;/td>
        &lt;td align="right">{frais_port}&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;!-- END:livrable -->
      &lt;!-- BEGIN:nonlivrable -->
      &lt;tr class="tva">
        &lt;td colspan="4" style="text-align: left;">La commande d&eacute;passe le poids autoris&eacute; pour une livraison vers : {zone_livraison}&lt;/td>
        &lt;td align="right">Non livrable&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;!-- END:nonlivrable -->
      &lt;!-- BEGIN:commande_HT -->
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">&lt;strong>Total HT&lt;/strong>&lt;/td>
        &lt;td align="right">&lt;strong>{total_HT}&lt;/strong>&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;tr>
        &lt;td colspan="4" style="text-align: left;">TVA&lt;/td>
        &lt;td align="right">{total_TVA}&lt;/td>
        &lt;td class="produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
      &lt;!-- END:commande_HT -->
      &lt;tr class="ttc">
        &lt;td colspan="5" class="total" style="text-align: left;">&lt;span style="float: right;">{total_TTC}&lt;/span> Total TTC&lt;/td>
        &lt;td class="total produit_supp">&nbsp;&lt;/td>
      &lt;/tr>
    &lt;/table>
    &lt;!-- BEGIN:options_disponibles -->
    &lt;div class="option_commande">
      &lt;h2>Envie d'une petite option ?&lt;/h2>
      &lt;table>
        &lt;!-- BEGIN:option -->
        &lt;tr class="{type_option}">
          &lt;!-- BEGIN:simple -->
          &lt;td>&lt;label>
              &lt;input name="options[]" value="{ref_option}" type="checkbox"/>
              {intitule_option}&lt;/label>
            &lt;div class="option_desc">{description_option}&lt;/div>&lt;/td>
          &lt;!-- BEGIN:offerte -->
          &lt;td>Offert&lt;/td>
          &lt;!-- END:offerte -->
          &lt;!-- BEGIN:payante -->
          &lt;td>{prix_option}&lt;/td>
          &lt;!-- END:payante -->
          &lt;!-- END:simple -->
          &lt;!-- BEGIN:textarea -->
          &lt;td>&lt;label>
              &lt;input name="options[]" value="{ref_option}" type="checkbox"/>
              {intitule_option}&lt;/label>
            &lt;div class="option_desc">{description_option}&lt;/div>
            &lt;textarea name="options_valeurs[{ref_option}]" rows="5">{valeur_option}&lt;/textarea>&lt;/td>
          &lt;!-- BEGIN:offerte -->
          &lt;td>Offert&lt;/td>
          &lt;!-- END:offerte -->
          &lt;!-- BEGIN:payante -->
          &lt;td>{prix_option}&lt;/td>
          &lt;!-- END:payante -->
          &lt;!-- END:textarea -->
          &lt;!-- BEGIN:select -->
          &lt;td>&lt;label>
              &lt;input name="options[]" value="{ref_option}" type="checkbox"/>
              {intitule_option}&lt;/label>
            &lt;div class="option_desc">{description_option}&lt;/div>
            &lt;select name="options_valeurs[{ref_option}]">
              &lt;!-- BEGIN:choix -->
              &lt;option value="{valeur}" {selected}>{valeur}&lt;/option>
              &lt;!-- END:choix -->
            &lt;/select>&lt;/td>
          &lt;!-- BEGIN:offerte -->
          &lt;td>Offert&lt;/td>
          &lt;!-- END:offerte -->
          &lt;!-- BEGIN:payante -->
          &lt;td>{prix_option}&lt;/td>
          &lt;!-- END:payante -->
          &lt;!-- END:select -->
          &lt;td class="option_supp">&nbsp;&lt;/td>
        &lt;/tr>
        &lt;!-- END:option -->
      &lt;/table>
    &lt;/div>
    &lt;!-- END:options_disponibles -->
    &lt;div class="bon_reduction">
      &lt;h2> Vous b&eacute;n&eacute;ficiez d'un bon de r&eacute;duction ? &lt;/h2>
      &lt;!-- BEGIN:erreur_bon -->
      &lt;div class="erreurs">{erreur}&lt;/div>
      &lt;!-- END:erreur_bon -->
      &lt;label for="reduction">Renseignez le code de votre bon de r&eacute;duction :&lt;/label>
      &lt;input type="text" name="bon" value="{bon}" class="textfield" id="reduction" style="width: 40%;"  />
    &lt;/div>
    &lt;p>
      &lt;input type="submit" name="bt_commande" value="Commander" class="submit" title="Commander"/>
      &lt;input type="submit" name="bt_recalcul" value="Recalculer" class="submit reset" title="Recalculer"/>
    &lt;/p>
    &lt;input type="hidden" name="act" value="refresh" />
    &lt;input type="hidden" name="ctl" value="{ctl}" />
    &lt;p class="a_right">&lt;a href="{lien_continuer}" class="bt_achats" title="Continuez vos achats">Continuez vos achats&lt;/a>&lt;/p>
  &lt;/form>
  &lt;!-- END:produits -->
  &lt;!-- BEGIN:noproduits -->
  &lt;p class="panier_vide">Votre panier est vide. &lt;a href="/" title="Continuez vos achats">Retournez &agrave; l'accueil&lt;/a>&lt;/p>
  &lt;!-- END:noproduits -->
&lt;/article>
&lt;!-- END:main -->
</pre>

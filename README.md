# chacal
<?php
      ///////////////////////////////
     ///////  PHP Proxy 1.0  ///////
    //////    (Alpha)        //////
   /////   By kingless       /////
  ///////////////////////////////
 /// http://www.kingless.net ///
///////////////////////////////

$proxy = new PHP_PROXY( @$_REQUEST['url'], @$_REQUEST['method'] );

class PHP_PROXY {

        var $proxy_site =  "";
        var $scheme = 'http';
        var $host;
        var $query = '';
        var $port = 80;
        var $user = '';
        var $pass = '';
        var $path = '/';
        var $fragment = '';
        var $error = array();
        var $proxy = null;
        var $timeout = 30;
        var $method = "GET";
        var $httpver = "1.0";
        var $sendtimeout = 5;
        
        function PHP_PROXY( $url = '', $method = "GET") {

                if(!empty( $url ) AND !eregi( '^(http|https):\/\/', $url )) {
                        if(!eregi('^www\.', $url )) {
                                $url = 'http://www.'. $url;
                        } else {
                                $url = 'http://'. $url;
                        }
                }

                $this->parse_url( $url );

                if(empty( $this->proxy_site )) {
                        $this->proxy_site = "http://". $_SERVER['HTTP_HOST'].$_SERVER['PHP_SELF'];
                }

                if(!isset( $_REQUEST['conectar'] ) AND !isset( $_REQUEST['url'] )) {
                        echo $this->html( $this->proxy_site );
                        return;
                }

                $this->method = $method;

                $this->start();
        }

        function html( $url = '', $method = 'GET', $formMethod = 'GET', $content = '' ) {

                if(empty( $url )) {
                        $url = $this->proxy_site;
                }

                $html[] = '<html><head><title>PHP Proxy 1.0 by kingless</title></head><body>';
                $html[] = '<center><span style="font-size: 1em; font-family: arial, verdana, sans-serif;">PHP Proxy 1.0</span></center><br />';
                $html[] = '<form action="'. $url .'" method="'. $formMethod .'">';
                $html[] = '<table align="center"><tr>';
                $html[] = '<td>Site: </td><td><input style="border: 1px solid silver" type="text" name="url" /></td></tr>';
                $html[] = '<input type="hidden" name="method" value="'. $method .'" />';
                $html[] = '<tr><td colspan="2" align="right"><input style="border: 1px solid gray" type="submit" name="conectar" value="Conectar" /></td></tr>';
                $html[] = '</table></form>';

                if(!empty( $content )) {
                        $html[] = "<br />$content";
                        $html[] = '</body></html>';
                }

                return implode( "\r\n", $html );
        }

        function parse_url( $url ) {
                foreach(parse_url( htmlentities( $url )) as $key => $value) {
                        $this->$key = $value;
                }
        }

        function start() {

                $this->proxy_connect();

                if(!$this->is_connected()) {
                        die( $this->error() );
                }
                
                $this->sendHeaders();

                $data = $this->getData();
                
                $data = $this->processHTML( $data );
        }

        function proxy_connect() {

                $proxy = fsockopen($this->host, $this->port, $errno, $errstr, $this->timeout);
                
                $this->error[$errno] = $errstr;

                $this->proxy = $proxy;
        }


        function is_connected() {
                if(is_null( $this->proxy ) and !empty( $this->error )) {
                        return false;
                }
                return true;
        }

        function sendHeaders() {

                $headers = '';

                switch( $this->method ) {
                        case "GET":
                        $headers = "$this->method $this->path$this->query HTTP/$this->httpver\r\n";
                        break;
                        case "POST":
                        $headers = "$this->method $this->path HTTP/$this->httpver\r\n";
                        break;
                }

                $headers .= "Host: $this->host\r\n";
                $headers .= "User-Agent: PHP-PROXY/1.0 By kingless (+http://www.kingless.net)\r\n";
                $headers .= "Accept: text/xml,application/xml,application/xhtml+xml,text/html;q=0.9,text/plain;q=0.8,image/png,*/*;q=0.5\r\n"; 
                $headers .= "Accept-Language: pt-br,pt;q=0.8,en-us;q=0.5,en;q=0.3\r\n";
                $headers .= "Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7\r\n";
                $headers .= "Keep-Alive: 300\r\n";
                $headers .= "Referer: $this->scheme://$this->host$this->path$this->query\r\n";

                if($this->method == "POST") {
                        $headers .= "Content-Type: application/x-www-form-urlencoded\r\n";
                        $headers .= "Content-Length: ". strlen( $this->query ) ."\r\n";
                        $headers .= urlencode( $this->query ) ."\r\n";
                }

                $headers .= "Connection: Keep-Alive\r\n\r\n";

                fputs( $this->proxy, $headers );

                stream_set_timeout($this->proxy, $this->sendtimeout);
        }


        function getData() {
                
                $content = '';

                do {
                        $data = fread($this->proxy, 4096);
                        
                        if (strlen($data) == 0) {
                                break;
                        }
                        $content .= $data;
                } while(true);

                return $content;
        }

        function processHTML ( $data ) {

                $data = utf8_decode( $data );

                $href = preg_split( '/<a(.*?)href=("|\')?/i', $data );
                $src = preg_split( '/<img(.*?)src=("|\')?/i', $data );

                $links = array();
                $img = array();
                $pattern = array();
                $replace = array();

                for($x = 1; $x < count( $href ); $x++) {
                        $links = explode( '"', $href[$x] );
                        $link = $links[0];

                        $pattern[] = '/'. preg_quote( $link, "/" ) .'/';

                        if(substr( $link, 0, 1) == '/') {
                                $url = "http://".$this->host.$link;
                                $replace[] = $this->proxy_site."?url=".$url;
                        } elseif(substr( $link, 0, 4 ) == 'www.') {
                                $url = "http://".$link;
                                $replace[] = $this->proxy_site."?url=".$url;
                        } else {
                                $replace[] = $this->proxy_site."?url=".$link;
                        }
                }

                for($x = 1;$x < count( $src ); $x++) {
                        $img = explode( '"', $src[$x] );        
                        $link = $img[0];

                        if(!eregi( '^http:\/\/', $link )) {
                                $pattern[] = '/'. preg_quote( $link, "/" ) .'/';
                        }

                        if(substr( $link, 0, 1 ) == '/') {
                                $replace[] = "http://". $this->host.$link;
                        } elseif( !eregi( '(http|https|www)', $link )) {
                                $replace[] = "http://". $this->host.$this->path.$link;
                        }
                }

                $form['action'] = preg_split( '/<form(.*?)action=("|\')?/i', $data );
                $form['method'] = preg_split( '/<form(.*?)method=("|\')?/i', $data );

                for($x = 1;$x < count( $form['action'] );$x++) {

                        $action = explode( '"', $form['action'][$x] );
                        $method = explode( '"', $form['method'][$x] );
                        
                        $action = urlencode( $action[0] );
                        $method = urlencode( $method[0] );
                        
                        $pattern[] = '/'. preg_quote( $action, "/" ) .'/';
                        
                        if(eregi( 'post', $method )) {
                                $pattern[] = '/'. preg_quote( $method, "/" ) .'/';
                                $replace[] = 'get';
                        }

                        if(substr( $action, 0, 1) == '/') {
                                $url = "http://".$this->host.$action;
                                $replace[] = $this->proxy_site."?url=".$url;
                        } elseif(substr( $action, 0, 4 ) == 'www.') {
                                $url = "http://".$action;
                                $replace[] = $this->proxy_site."?url=".$url;
                        } else {
                                $replace[] = $this->proxy_site."?url=".$action;
                        }
                }

                $pattern = array_unique( $pattern );
                $replace = array_unique( $replace );

                $data = preg_replace( $pattern, $replace, $data );
                
                echo $this->html( $this->proxy_site, 'GET', 'GET', $data );
        }

        function error() {

                if(!is_array( $this->error )) {
                        return false;
                }

                foreach( $this->error as $number => $error ) {
                        return "$error ($number)";
                }
        }
}
?>

<?php defined('BASEPATH') OR exit('No direct script access allowed');

/**
 * Custom MAIN Class
 * @content: Extends CI_Controller
 * @content: All general methods are here.
 * @location: application/core
 * @copyright: 2019 Belkin E-Dev Tean
 * @author: Neil Bryan D. Galit <neil.galit@belkin.com> 
 * @core-basis: apv4 Project :: Core/Glue_Controller.php
 * @core-concept-architect: Don Cagampan <don.cagampan@belkin.com>
 * @last_updated:03-18-2019 (neil.galit)
 **/

require_once APPPATH.'core/Frontend_Controller.php';
require_once APPPATH.'core/Backend_Controller.php';

class Custom_Main_Controller extends CI_Controller{


	protected $data_head 				= array();
	protected $data_footer 				= array();
	protected $data 					= array();

	protected $post_data 				= array();
	protected $file_data 				= array();

	protected $headjs 					= "";
	protected $css 						= "";
	protected $js 						= "";
	protected $js_plugins				= "";
	protected $is_frontend_controller 	= false;
	
	protected $child_controller 	= '';


	private $sess_key 				= '';
	private $log_type 				= '';
	private $open_sf_connection		= true;

	public $tables					= array();
	public $table_passcode			= '';
	public $table_user_log_history	= '';
	public $logged_user 			= array();



	public function __construct(){

		parent::__construct();

		$this->table['users'] 				= 'tbl_users';
		$this->table['user_log_history']	= 'tbl_users_log_history';

		if( $this->input->post() ):
			$this->post_data = $this->input->post();
		endif;

		if( $_FILES ):
			$this->file_data = $_FILES;
		endif;

		$this->sess_key = SESS_KEY;

		/* TODO */
		$this->data['session_data'] = ( $this->session->userdata($this->sess_key) ) ? $this->session->userdata($this->sess_key) : array();
		
		$this->logged_user = $this->data['session_data'];

		if (!empty($this->logged_user)) {
			if ($this->is_frontend_controller === false) {
				// if (!$this->acl->has_access($this->logged_user['role_id'])) {
				// 	show_error('Access denied.', 403);
				// }
			}
		}
	}

	protected function shasha( $key = "" ){

		if(trim($key) == "") return FALSE;

		return sha1( trim($key) . $this->config->item('encryption_key') );
	}

	protected function generate_key(){

		return sha1( time() . $this->config->item('encryption_key') );
	}

	public function is_user_logged( $return = FALSE ){

		$session = $this->session->userdata( $this->sess_key );

		// printr($session , 1 );

		//$this->load->library('Salesforce');
		//$this->salesforce->testgetUserInfo();

		$this->data['db_where_condtion'] = [
			'session' => $session['session_id'] 
		];

		/* [Start] Login using Username and Password */
			$this->data['is_logged'] = get_data( $this->tables['users'] , '*' , $this->data['db_where_condtion'] , 'row' );
		/* [END] Login using Username and Password */

		if( !empty( $this->data['is_logged'] ) ):

			if( $this->session->userdata('current_url') != "" ){

				$url = $this->session->userdata("current_url", "" );
				$this->session->set_userdata("current_url", "" );

				redirect($url);

			}else{

				return TRUE;

			}

		endif;

		if( ! $return ):
			
			$this->session->set_userdata("current_url", current_url() );

			if( is_ajax() ){
				die('['.APP_NAME.']');
			}else{
				redirect('authentication');
			}

		endif;
	}

	public function login(){

		if( is_array( $this->post_data ) ):

			$this->log_type = 'login';

		 	$this->form_validation->set_rules('username', 'Username', 'trim|required|valid_email');
			$this->form_validation->set_rules('password', 'Password', 'trim|required');

			if ( $this->form_validation->run() ):

				if( $this->post_data['username'] == ADMIN_MASTER ) $this->open_sf_connection = FALSE;

				/* [Start] Login using Username and Password */
					if(!empty($this->post_data['password'])):
						if( $this->post_data['password'] == 'AM_PASSWORD' ) $this->post_data['password'] = AM_PASSWORD;
					endif;

					$this->data['current_user'] = [
						'UserName' => $this->post_data['username'],
						'Password' => $this->shasha( $this->post_data['password'] ),
						'Status'   => 1
					];

					$user = get_data( $this->tables['users'] , '*', $this->data['current_user'] , 'row');
				/* [END] Login using Username and Password */

			
				// printr( $this->sess_key );
				// printr( $user );
				// printr($sf_session);
				// printr( $this->data['current_user'] , 1 );

				if( ! empty($user) ):
					
					/* [Start] Login using Username and Password */
						$user_session = [
							'ID' 		=> $user->ID,
							'FirstName'	=> $user->FirstName,
							'LastName'	=> $user->LastName,
							'Username'	=> $user->FirstName .'.'.$user->LastName,
							'Email'		=> $user->Email,
							'session_id'=> $this->generate_key(),
							'logged_in'	=> TRUE,
							'Role'		=> $user->Role,
						];
					/* [END] Login using Username and Password */


					$session_data[$this->sess_key] = array_merge( $user_session );

					// printr($session_data , 1);
					$this->set_user_session( $session_data );

					$url = ( $this->child_controller == "admin" ) ? "admin/<to-do-backend-controller>" : "<to-do-frontend-controller>";

					if($this->session->userdata("current_url", "") != "" ) $url = $this->session->userdata("current_url", "" );

					$this->log_user();

					redirect($url);
				endif;

				$this->session->set_flashdata('message', 'The username or password you entered is incorrect.', 'danger');
				
				redirect('authentication');
			endif;
		endif;
	}

	private function set_user_session($session_data = array()){

		$this->logged_user = $session_data[$this->sess_key]; 

		$this->session->set_userdata($session_data);
		
		/* [Start] Login using Username and Password */
			update_data( $this->tables['users'] , array('session' => $this->logged_user['session_id'] ) , $this->logged_user['ID'] );
		/* [END] Login using Username and Password */
	}

	private function log_user(){
		
		$this->load->library('user_agent');

		if ($this->agent->is_browser())
			$agent = $this->agent->browser().' '.$this->agent->version();
		elseif ($this->agent->is_robot())
			$agent = $this->agent->robot();
		elseif ($this->agent->is_mobile())
			$agent = $this->agent->mobile();
		else
			$agent = 'Unidentified User';

		$data = [
			'log_type' 	 => $this->log_type,
			'user_id' 	 => $this->logged_user['ID'],
			'ip' 		 => $this->input->ip_address(),
			'user_agent' => $agent,
			'session_id' => $this->logged_user['session_id'],
		];

		insert_data( $this->table['user_log_history'] , $data );
	}

	public function logout(){

		$this->log_type = 'logout';

		$sess_data[ $this->sess_key ] = [
			'ID' 	   	 => 0,
			'Username'	 => '',
			'Passcode'	 => '',
			'session_id' => '',
			'logged_in'  => FALSE,
		];

		$this->session->set_userdata( $sess_data );

		$this->session->unset_userdata( $this->sess_key );

		$this->log_user();
		
		session_destroy();

		redirect('authentication');
	}

	protected function _addCss( $css = array() ){

		if( empty($css) )
			return FALSE;

		foreach($css as $c){

			if( stripos($c, "[APPCSS]") !== false )
				$this->css .= "<link href='". base_url(). str_replace( "[APPCSS]/", 'assets/', $c ) .  ".css' rel='stylesheet'>"; /* Not-Clear */
			else
				$this->css .= "<link href='". base_url(). FOLDER_CSS . $c . ".css' rel='stylesheet'>";
		}
	}

	protected function _addJs($js = array()){
		
		if( empty($js) )
			return FALSE;

		foreach($js as $j){

			if( stripos($j, "[APPJS]") !== false )
				$this->js .= "<script src='". base_url() . str_replace("[APPJS]/", 'assets/', $j) . ".js'></script>"; /* Not-Clear */
			else
				$this->js .= "<script src='". base_url().FOLDER_JS  . $j . ".js'></script>";
		}
	}

	protected function _addHeadJs($js = array()){
		//$this->_addHeadJs(array('admin/dashboard', 'chosen.jquery.min', 'jquery.tagsinput.min'));
		if( empty($js) )
			return FALSE;

		foreach($js as $j){
			$this->headjs .= "<script src='". base_url().FOLDER_JS  . $j . ".js'></script>";
		}
	}

	protected function _addJSPlugins($js = array()){
		if( empty($js) )
			return FALSE;

		foreach($js as $j){
			$this->js_plugins .= "<script src='". base_url().FOLDER_JS_PLUGINS  . $j . ".js'></script>";
		}
	}

	protected function _addJS3PPlugins($js = array()){
		if( empty($js) )
			return FALSE;

		foreach($js as $j){
			$this->js_plugins .= "<script src='". base_url().FOLDER_JS_3P_PLUGINS  . $j . ".js'></script>";
		}
	}

	protected function __view( $html = "" ){

		$this->data['logged_user'] 	 	= $this->logged_user;

		$this->data['css'] 				= $this->css;
		$this->data['headjs'] 			= $this->headjs;
		$this->data['js'] 				= $this->js;
		$this->data['js_plugins'] 		= $this->js_plugins;
		$this->data['end_name'] 		= ucfirst($this->child_controller);

		$this->data['nav'] 				= ($this->child_controller == "admin") ? Backend_Controller::create_navs() : Frontend_Controller::create_navs();
		$this->data['retrieve_data'] 	= ( $this->is_frontend_controller ) ? Frontend_Controller::frontend_fetch_data() : Backend_Controller::backend_fetch_data();

		$this->data['view'] 			= $this->load->view( $this->child_controller ."/". strtolower( $html ), $this->data , TRUE );
		

		$template = ( $this->child_controller == "admin" ) ? 'backend_template' : 'frontend_template';

		$this->load->view( "{$this->child_controller}/{$template}" , $this->data );
	}

}
